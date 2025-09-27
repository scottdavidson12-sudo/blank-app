import os, json
from datetime import datetime, timedelta, timezone
import pandas as pd
import requests
import streamlit as st
from dateutil import tz

# ---------- Look & feel ----------
st.set_page_config(page_title="Cosy — Heat Pump", page_icon="🔥", layout="wide")
st.markdown("""
<style>
/* Tesla-ish dark UI */
:root { --card-bg: rgba(255,255,255,0.06); --blur: blur(12px); }
.block-container { padding-top: 1rem; }
header, footer { visibility: hidden; }

/* Glass cards */
.cosy-card {
  background: var(--card-bg);
  backdrop-filter: var(--blur);
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 16px;
  padding: 16px 18px;
  margin-bottom: 12px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.25);
}
.cosy-title { font-size: 1.2rem; font-weight: 600; margin-bottom: 8px; }
.kpi { display:flex; justify-content:space-between; align-items:center; padding: 10px 0; border-bottom: 1px dashed rgba(255,255,255,0.08); }
.kpi:last-child { border-bottom: none; }
.kpi .label { opacity:.8; }
.kpi .value { font-weight:600; font-size:1.05rem; }

/* Buttons */
div.stButton > button {
  width: 100%; border-radius: 12px; padding: .7rem 1rem; border: 1px solid rgba(255,255,255,.15);
  background: linear-gradient(180deg, rgba(255,255,255,.12), rgba(255,255,255,.06));
}
</style>
""", unsafe_allow_html=True)

# ---------- Timezones ----------
UTC = timezone.utc
LONDON = tz.gettz("Europe/London")

# ---------- API ----------
GQL_URL = "https://api.octopus.energy/v1/graphql/"
REST_BASE = "https://api.octopus.energy/v1"

def obtain_token(api_key:str) -> str:
    q = {"query": "mutation($k:String!){ obtainKrakenToken(input:{APIKey:$k}){ token } }",
         "variables": {"k": api_key}}
    r = requests.post(GQL_URL, json=q, timeout=20)
    r.raise_for_status()
    data = r.json()
    return data["data"]["obtainKrakenToken"]["token"]

def gql(query:str, variables:dict, token:str):
    h = {"Authorization": f"Bearer {token}"}
    r = requests.post(GQL_URL, headers=h, json={"query": query, "variables": variables}, timeout=20)
    r.raise_for_status()
    j = r.json()
    if "errors" in j:
        raise RuntimeError(j["errors"])
    return j["data"]

def list_euids(account:str, token:str):
    q = "query($a:String){ octoHeatPumpControllerEuids(accountNumber:$a) }"
    return gql(q, {"a": account}, token)["octoHeatPumpControllerEuids"] or []

def live_perf(euid:str, token:str):
    q = """
    query($e:ID!){
      octoHeatPumpLivePerformance(euid:$e){
        readAt coefficientOfPerformance
        powerInput{value unit} heatOutput{value unit} outdoorTemperature{value unit}
      }}"""
    return gql(q, {"e": euid}, token)["octoHeatPumpLivePerformance"]

def controller_status(account:str, euid:str, token:str):
    q = """
    query($a:String!, $e:ID!){
      octoHeatPumpControllerStatus(accountNumber:$a, euid:$e){
        sensors{name value unit}
        zones{name callingForHeat temperature{value unit}}
      }}"""
    return gql(q, {"a": account, "e": euid}, token)["octoHeatPumpControllerStatus"]

def timeseries(euid:str, start_iso:str, end_iso:str, grouping:str, token:str):
    q = """
    query($e:ID!, $s:DateTime!, $t:DateTime!, $g:PerformanceGrouping!){
      octoHeatPumpTimeSeriesPerformance(euid:$e, startAt:$s, endAt:$t, performanceGrouping:$g){
        startAt endAt energyInput{value unit} energyOutput{value unit}
      }}"""
    rows = gql(q, {"e": euid, "s": start_iso, "t": end_iso, "g": grouping}, token)["octoHeatPumpTimeSeriesPerformance"]
    df = pd.DataFrame([{
        "startAt": pd.to_datetime(r["startAt"]),
        "kWh_in": r["energyInput"]["value"],
        "kWh_out": r["energyOutput"]["value"]
    } for r in rows]).sort_values("startAt")
    if df.empty: return df
    df["COP"] = (df["kWh_out"] / df["kWh_in"]).replace([pd.NA, float("inf")], pd.NA)
    return df

def current_tariff_from_account(account:str, api_key:str):
    r = requests.get(f"{REST_BASE}/accounts/{account}/", auth=(api_key,""), timeout=20)
    r.raise_for_status()
    acc = r.json()
    for agr in acc.get("electricity_agreements", []):
        if agr.get("valid_to") is None:
            tariff = agr["tariff_code"]
            product = tariff.split("-E-")[0]
            return product, tariff
    return None, None

def cosy_rates(product:str, tariff:str, api_key:str):
    now = datetime.now(UTC)
    start = now.replace(hour=0, minute=0, second=0, microsecond=0)
    end = start + timedelta(days=2)
    params = {"period_from": start.isoformat().replace("+00:00","Z"),
              "period_to": end.isoformat().replace("+00:00","Z"),
              "order_by": "valid_from", "page_size": 250}
    r = requests.get(f"{REST_BASE}/products/{product}/electricity-tariffs/{tariff}/standard-unit-rates/",
                     params=params, auth=(api_key,""), timeout=20)
    r.raise_for_status()
    rows = r.json().get("results", [])
    if not rows: return pd.DataFrame()
    df = pd.DataFrame([{
        "from": pd.to_datetime(x["valid_from"]).tz_convert(LONDON),
        "to": pd.to_datetime(x["valid_to"]).tz_convert(LONDON),
        "p_per_kWh": x["value_inc_vat"]
    } for x in rows])
    return df

# ---------- UI: sign-in ----------
st.markdown("<div class='cosy-card'><div class='cosy-title'>Cosy — Heat Pump</div>"
            "<small>Paste your Octopus API key and account to preview the app. Nothing leaves your device except calls to Octopus.</small></div>", unsafe_allow_html=True)

if "api_key" not in st.session_state:
    st.session_state.api_key = ""
if "account" not in st.session_state:
    st.session_state.account = "A-2E5EC985"
if "token" not in st.session_state:
    st.session_state.token = None
if "selected_euid" not in st.session_state:
    st.session_state.selected_euid = None

with st.form("login", clear_on_submit=False):
    api_key = st.text_input("API key", value=st.session_state.api_key, type="password", help="Find this in the Octopus account portal.")
    account = st.text_input("Account number", value=st.session_state.account)
    submitted = st.form_submit_button("Connect")

if submitted:
    try:
        st.session_state.api_key = api_key.strip()
        st.session_state.account = account.strip()
        st.session_state.token = obtain_token(st.session_state.api_key)
        euids = list_euids(st.session_state.account, st.session_state.token)
        if not euids: st.error("No heat-pump controllers (EUID) found for this account."); st.stop()
        st.session_state.selected_euid = euids[0]
        st.success("Connected!")
    except Exception as e:
        st.error(f"Couldn't connect: {e}")

if not st.session_state.token:
    st.stop()

# ---------- Dashboard ----------
euid = st.selectbox("Heat pump", options=[st.session_state.selected_euid], index=0)
colA, colB = st.columns(2)

with colA:
    st.markdown("<div class='cosy-card'><div class='cosy-title'>Live</div>", unsafe_allow_html=True)
    live = live_perf(euid, st.session_state.token)
    def _fmt(x, p=2):
        try: return f"{float(x):.{p}f}"
        except: return "—"
    st.markdown(f"""
    <div class='kpi'><span class='label'>Live COP</span><span class='value'>{_fmt(live.get('coefficientOfPerformance'),2)}</span></div>
    <div class='kpi'><span class='label'>Power in</span><span class='value'>{_fmt(live.get('powerInput',{{}}).get('value'))} {live.get('powerInput',{{}}).get('unit','')}</span></div>
    <div class='kpi'><span class='label'>Heat out</span><span class='value'>{_fmt(live.get('heatOutput',{{}}).get('value'))} {live.get('heatOutput',{{}}).get('unit','')}</span></div>
    <div class='kpi'><span class='label'>Outdoor</span><span class='value'>{_fmt(live.get('outdoorTemperature',{{}}).get('value'),1)} {live.get('outdoorTemperature',{{}}).get('unit','')}</span></div>
    """, unsafe_allow_html=True)
    if live.get("readAt"):
        when = pd.to_datetime(live["readAt"]).tz_convert(LONDON).strftime("%H:%M:%S %Z")
        st.caption(f"Updated {when}")
    st.markdown("</div>", unsafe_allow_html=True)

with colB:
    st.markdown("<div class='cosy-card'><div class='cosy-title'>Zones & Sensors</div>", unsafe_allow_html=True)
    status = controller_status(st.session_state.account, euid, st.session_state.token)
    zdf = pd.DataFrame(status.get("zones", []))
    if not zdf.empty:
        zdf["callingForHeat"] = zdf["callingForHeat"].map(lambda x: "Heating" if x else "Idle")
        zdf["temp"] = zdf["temperature"].map(lambda t: f"{t['value']:.1f} {t['unit']}" if isinstance(t, dict) else "—")
        st.dataframe(zdf[["name","callingForHeat","temp"]].rename(columns={"name":"Zone"}), hide_index=True, use_container_width=True)
    sdf = pd.DataFrame(status.get("sensors", []))
    if not sdf.empty:
        sdf["reading"] = sdf.apply(lambda r: f"{r.get('value')} {r.get('unit','')}", axis=1)
        st.dataframe(sdf[["name","reading"]].rename(columns={"name":"Sensor"}), hide_index=True, use_container_width=True)
    st.markdown("</div>", unsafe_allow_html=True)

st.markdown("<div class='cosy-card'><div class='cosy-title'>History</div>", unsafe_allow_html=True)
now = datetime.now(UTC)
start = now - timedelta(days=7)
df = timeseries(euid, start.isoformat(), now.isoformat(), "HOUR", st.session_state.token)
if df.empty:
    st.info("No historical data returned for the last 7 days.")
else:
    st.line_chart(df.set_index("startAt")[["kWh_in","kWh_out"]])
    st.line_chart(df.set_index("startAt")[["COP"]])
st.markdown("</div>", unsafe_allow_html=True)

st.markdown("<div class='cosy-card'><div class='cosy-title'>Cosy tariff windows</div>", unsafe_allow_html=True)
product, tariff = current_tariff_from_account(st.session_state.account, st.session_state.api_key)
if product and tariff:
    rates = cosy_rates(product, tariff, st.session_state.api_key)
    if not rates.empty:
        st.dataframe(rates.rename(columns={"from":"From","to":"To","p_per_kWh":"p/kWh inc VAT"}), hide_index=True, use_container_width=True)
        st.bar_chart(rates.set_index("From")["p_per_kWh"])
    else:
        st.info("No rates for today/tomorrow.")
else:
    st.info("No active electricity tariff found.")
st.markdown("</div>", unsafe_allow_html=True)
