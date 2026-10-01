# Previsor-futebol-
App de previsão de futebol 
import streamlit as st
import requests
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
from datetime import datetime
import os

API_KEY = os.getenv("FOOTBALL_DATA_TOKEN")
st.set_page_config(page_title="Previsor de Futebol", page_icon="⚽", layout="wide")

st.title("⚽ Previsor Global de Futebol")
st.markdown("*Análise com dados reais, H2H e 
BASE_URL = "https://v3.football.api-sports.io"
HEADERS = {'x-rapidapi-key': API_KEY, 'x-rapidapi-host': 'v3.football.api-sports.io'}

@st.cache_data(ttl=3600, show_spinner=False)
def api_get(endpoint, params=None):
    if params is None: params = {}
    try:
        r = requests.get(f"{BASE_URL}/{endpoint}", headers=HEADERS, params=params, timeout=20)
        r.raise_for_status()
        data = r.json()
        if data.get('errors') and len(data['errors']) > 0: return None
        return data.get('response', [])
    except: return None

@st.cache_data(ttl=86400, show_spinner=False)
def get_paises():
    d = api_get("countries")
    return sorted([p['name'] for p in d]) if d else []

@st.cache_data(ttl=86400, show_spinner=False)
def get_ligas(pais):
    d = api_get("leagues", {"country": pais, "current": "true"})
    return d if d else []

@st.cache_data(ttl=3600, show_spinner=False)
def get_times(liga_id, temp):
    d = api_get("teams", {"league": liga_id, "season": temp})
    return d if d else []

@st.cache_data(ttl=600, show_spinner=False)
def get_proxima(time_id):
    d = api_get("fixtures", {"team": time_id, "next": 1})
    return d[0] if d else None

@st.cache_data(ttl=3600, show_spinner=False)
def get_previsao(fid):
    d = api_get("predictions", {"fixture": fid})
    return d[0] if d else None

@st.cache_data(ttl=3600, show_spinner=False)
def get_h2h(t1, t2):
    d = api_get("fixtures/headtohead", {"h2h": f"{t1}-{t2}", "last": 10})
    return d if d else []

@st.cache_data(ttl=3600, show_spinner=False)
def get_ultimos(time_id, q=5):
    d = api_get("fixtures", {"team": time_id, "last": q})
    return d if d else []

@st.cache_data(ttl=3600, show_spinner=False)
def get_classif(liga_id, temp):
    d = api_get("standings", {"league": liga_id, "season": temp})
    return d[0] if d else None

def analisar_h2h(jogos):
    if not jogos: return None
    vc = vf = e = gt = 0
    for j in jogos:
        gc = j['goals']['home'] or 0
        gf = j['goals']['away'] or 0
        gt += gc + gf
        if gc > gf: vc += 1
        elif gf > gc: vf += 1
        else: e += 1
    t = len(jogos)
    return {"total": t, "vc": vc, "vf": vf, "e": e,
            "media": round(gt/t, 2) if t else 0}

def analisar_ultimos(jogos, tid):
    if not jogos: return None
    v = e = d = gm = gs = 0
    for j in jogos:
        cid = j['teams']['home']['id']
        gc = j['goals']['home'] or 0
        gf = j['goals']['away'] or 0
        if cid == tid:
            gm += gc; gs += gf
            if gc > gf: v += 1
            elif gc < gf: d += 1
            else: e += 1
        else:
            gm += gf; gs += gc
            if gf > gc: v += 1
            elif gf < gc: d += 1
            else: e += 1
    t = len(jogos)
    return {"v": v, "e": e, "d": d, "gm": gm, "gs": gs,
            "mm": round(gm/t, 2) if t else 0,
            "ms": round(gs/t, 2) if t else 0,
            "pts": v*3+e, "total": t}

with st.sidebar:
    st.header("🔍 Selecione a Partida")
    paises = get_paises()
    if not paises:
        st.error("⚠️ Erro ao carregar. Verifique a API_KEY nos secrets.")
        st.stop()
    idx = paises.index("Brazil") if "Brazil" in paises else 0
    pais = st.selectbox("🌍 País", paises, index=idx)
    ligas = get_ligas(pais)
    if not ligas:
        st.warning(f"Sem ligas para {pais}"); st.stop()
    lo = {}
    for l in ligas:
        s = next((x for x in l['seasons'] if x.get('current')), None)
        if s: lo[f"{l['league']['name']} ({s['year']})"] = (l['league']['id'], s['year'])
    if not lo:
        st.warning("Sem ligas ativas"); st.stop()
    lnome = st.selectbox("🏆 Liga", list(lo.keys()))
    lid, temp = lo[lnome]
    times = get_times(lid, temp)
    if not times:
        st.warning("Sem times"); st.stop()
    td = {t['team']['name']: t['team']['id'] for t in sorted(times, key=lambda x: x['team']['name'])}
    tnome = st.selectbox("⚽ Time", list(td.keys()))
    tid = td[tnome]
    buscar = st.button("🔮 Analisar Partida", type="primary", use_container_width=True)

if buscar:
    with st.spinner("🔄 Buscando dados..."):
        prox = get_proxima(tid)
        if not prox:
            st.warning(f"Sem partidas futuras para {tnome}"); st.stop()
        fid = prox['fixture']['id']
        cid = prox['teams']['home']['id']; fid_a = prox['teams']['away']['id']
        casa = prox['teams']['home']['name']; fora = prox['teams']['away']['name']
        data = prox['fixture']['date']
        est = prox['fixture']['venue'].get('name', 'N/A')
        liga_j = prox['league']['name']
        rodada = prox['league']['round']

        st.markdown(f"### 🏟️ {casa} vs {fora}")
        st.caption(f"{liga_j} • {rodada} | 📅 {data[:16].replace('T',' ')} | 📍 {est}")
        st.markdown("---")

        prev = get_previsao(fid)
        h2h = get_h2h(cid, fid_a)
        uc = get_ultimos(cid, 5); uf = get_ultimos(fid_a, 5)

        if prev:
            p = prev['predictions']['percent']
            pc = int(p['home'].replace('%','')); pe = int(p['draw'].replace('%',''))
            pf = int(p['away'].replace('%',''))

            st.markdown("## 🎯 Probabilidades")
            c1, c2, c3 = st.columns(3)
            c1.metric(f"🏠 {casa}", f"{pc}%")
            c2.metric("🤝 Empate", f"{pe}%")
            c3.metric(f"✈️ {fora}", f"{pf}%")

            fig = go.Figure()
            fig.add_trace(go.Bar(y=[""], x=[pc], name=casa, orientation='h', marker_color='#2ecc71', text=f"{pc}%", textposition='inside'))
            fig.add_trace(go.Bar(y=[""], x=[pe], name="Empate", orientation='h', marker_color='#f39c12', text=f"{pe}%", textposition='inside'))
            fig.add_trace(go.Bar(y=[""], x=[pf], name=fora, orientation='h', marker_color='#e74c3c', text=f"{pf}%", textposition='inside'))
            fig.update_layout(barmode='stack', height=120, margin=dict(l=0,r=0,t=0,b=0))
            st.plotly_chart(fig, use_container_width=True)

            placar = prev['predictions']['goals']
            st.info(f"🎯 **Placar previsto:** {placar['home']} - {placar['away']}")

            st.markdown("### 💡 Recomendação da IA")
            st.success(prev['predictions'].get('advice', 'Sem recomendação'))

            st.markdown("## 📊 Comparativo")
            comp = prev['comparison']
            df = pd.DataFrame({
                "Métrica": ["Forma", "Ataque", "Defesa", "Posse"],
                casa: [comp['form']['home'], comp['att']['home'], comp['def']['home'], comp['possession']['home']],
                fora: [comp['form']['away'], comp['att']['away'], comp['def']['away'], comp['possession']['away']]
            })
            st.dataframe(df, use_container_width=True, hide_index=True)

        st.markdown("---")
        st.markdown("## ⚔️ H2H (últimos 10)")
        hs = analisar_h2h(h2h)
        if hs and hs['total'] > 0:
            c1, c2, c3, c4 = st.columns(4)
            c1.metric(f"Vit. {casa}", hs['vc'])
            c2.metric("Empates", hs['e'])
            c3.metric(f"Vit. {fora}", hs['vf'])
            c4.metric("Média Gols", hs['media'])
            fig2 = px.pie(values=[hs['vc'], hs['e'], hs['vf']], names=[casa, "Empates", fora],
                          color_discrete_sequence=['#2ecc71','#f39c12','#e74c3c'], hole=0.4)
            st.plotly_chart(fig2, use_container_width=True)
        else:
            st.info("Sem histórico disponível.")

        st.markdown("---")
        st.markdown("## 📅 Forma Recente")
        col1, col2 = st.columns(2)
        with col1:
            st.markdown(f"### 🏠 {casa}")
            sc = analisar_ultimos(uc, cid)
            if sc:
                st.write(f"✅ {sc['v']}V | ⚖️ {sc['e']}E | ❌ {sc['d']}D")
                st.write(f"⚽ Marcados: {sc['gm']} (média {sc['mm']})")
                st.write(f"🥅 Sofridos: {sc['gs']} (média {sc['ms']})")
        with col2:
            st.markdown(f"### ✈️ {fora}")
            sf = analisar_ultimos(uf, fid_a)
            if sf:
                st.write(f"✅ {sf['v']}V | ⚖️ {sf['e']}E | ❌ {sf['d']}D")
                st.write(f"⚽ Marcados: {sf['gm']} (média {sf['mm']})")
                st.write(f"🥅 Sofridos: {sf['gs']} (média {sf['ms']})")

        st.markdown("---")
        st.markdown(f"## 🏆 Classificação - {liga_j}")
        cl = get_classif(prox['league']['id'], prox['league']['season'])
        if cl and cl.get('league', {}).get('standings'):
            st_data = cl['league']['standings'][0]
            tab = [{"Pos": t['rank'], "Time": t['team']['name'], "Pts": t['points'],
                    "J": t['all']['played'], "V": t['all']['win'], "E": t['all']['draw'],
                    "D": t['all']['lose'], "SG": t['goalsDiff']} for t in st_data]
            st.dataframe(pd.DataFrame(tab), use_container_width=True, hide_index=True, height=400)

        st.caption(f"🕐 Atualizado: {datetime.now().strftime('%d/%m/%Y %H:%M')}")
else:
    st.markdown("""
    ### 👋 Bem-vindo!
    Selecione **país → liga → time** na barra lateral e clique em **Analisar Partida**.

    O app vai buscar automaticamente:
    - 🎯 Probabilidades reais
    - ⚔️ Confrontos diretos (H2H)
    - 📅 Forma recente
    - 🏆 Classificação da liga
    - 💡 Recomendação da IA
    """)
