# bot_binario_preco_volume.py

import pandas as pd
import numpy as np
import streamlit as st
import datetime as dt
import random
from streamlit_extras.metric_cards import style_metric_cards

# Função para simular dados de candles (para teste)
def gerar_candles_fake(qtd=50):
    candles = []
    timestamp = dt.datetime.now()
    preco = 1.1000
    for _ in range(qtd):
        open_ = preco
        high = open_ + round(random.uniform(0.0002, 0.0010), 5)
        low = open_ - round(random.uniform(0.0002, 0.0010), 5)
        close = round(random.uniform(low, high), 5)
        volume = random.randint(50, 500)
        candles.append({
            'timestamp': timestamp.strftime('%H:%M:%S'),
            'open': open_, 'high': high, 'low': low, 'close': close, 'volume': volume
        })
        timestamp -= dt.timedelta(minutes=1)
        preco = close
    return pd.DataFrame(candles[::-1])

# Estratégia de sinal com base em preço e volume
def gerar_sinais(df):
    sinais = []
    media_volume = df['volume'].rolling(window=20).mean()
    for i in range(20, len(df)):
        vol = df['volume'].iloc[i]
        candle = df.iloc[i]
        anterior = df.iloc[i-1]

        if vol > 1.5 * media_volume.iloc[i]:
            if candle['high'] > anterior['high'] and candle['close'] < candle['open']:
                sinais.append((candle['timestamp'], '🔴 PUT', 'Volume alto + candle de reversão (venda)'))
            elif candle['low'] < anterior['low'] and candle['close'] > candle['open']:
                sinais.append((candle['timestamp'], '🟢 CALL', 'Volume alto + candle de reversão (compra)'))
    return sinais

# Interface Web com Streamlit
st.set_page_config(page_title="Bot de Sinais Binários", layout="centered")
st.markdown("""
<style>
    .main { background-color: #f7f9fa; }
    h1, h2, h3 { color: #202c39; }
    .stButton > button { background-color: #007bff; color: white; border-radius: 10px; padding: 0.5em 1em; }
    .stAlert { border-radius: 10px; }
</style>
""", unsafe_allow_html=True)

st.title("🤖 Bot de Sinais Binários")
st.markdown("""
Este bot gera **sinais de entrada (CALL/PUT)** com base em análise de **preço e volume**.
A estratégia identifica clímax de volume e padrões de reversão de candle para oferecer sinais mais assertivos.
""")

# Sidebar
st.sidebar.header("⚙️ Configurações")
timeframe = st.sidebar.selectbox("Timeframe", options=["M1", "M5"], index=0)
modo = st.sidebar.radio("Modo", ["Simulação", "Tempo real"], index=0)
st.sidebar.markdown("---")
if st.sidebar.button("🔄 Atualizar Candles"):
    df = gerar_candles_fake()
    st.session_state['df'] = df
else:
    df = st.session_state.get('df', gerar_candles_fake())

# Gráficos
st.subheader("📉 Gráfico de Preço")
st.line_chart(df[['close']])

st.subheader("📊 Volume")
st.bar_chart(df[['volume']])

# Geração de sinais
sinais = gerar_sinais(df)

st.subheader("📢 Sinais Detectados")
if sinais:
    for ts, direcao, motivo in sinais:
        st.success(f"[{ts}] {direcao} - {motivo}")
else:
    st.info("Nenhum sinal detectado com os critérios atuais.")

# Créditos
st.markdown("---")
st.caption("Desenvolvido com 💻 por ChatGPT | Estratégia: Preço + Volume")
