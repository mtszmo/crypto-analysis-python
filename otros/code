import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.ticker as mtick
import glob
import seaborn as sns
from scipy.stats import linregress
import os



sns.set_theme(style="whitegrid")

ruta = 'archive'
archivos_csv = glob.glob(os.path.join(ruta, "*.csv"))

lista_dataframes = []

for archivo in archivos_csv:
    df_temporal = pd.read_csv(archivo)
    
    nombre_archivo = os.path.basename(archivo)
    simbolo = nombre_archivo.split('-')[0] 
    df_temporal['Simbolo'] = simbolo
    
    lista_dataframes.append(df_temporal)

df_completo = pd.concat(lista_dataframes, ignore_index=True)


df_completo['Date'] = pd.to_datetime(df_completo['Date']).dt.tz_localize(None)

columnas_a_eliminar = ['Dividends', 'Stock Splits']
df_completo = df_completo.drop(columns=[col for col in columnas_a_eliminar if col in df_completo.columns])

df_completo = df_completo.sort_values(by=['Simbolo', 'Date']).reset_index(drop=True)

print(df_completo.info())
print("\n")
print(df_completo.head())

fecha_inicio = '2021-01-01'
df_filtrado = df_completo[df_completo['Date'] >= fecha_inicio].copy()

print(f"Datos filtrados desde: {df_filtrado['Date'].min()} hasta {df_filtrado['Date'].max()}\n")

def calcular_roi(grupo):
    precio_inicial = grupo['Close'].iloc[0] 
    precio_final = grupo['Close'].iloc[-1]   
    roi = ((precio_final - precio_inicial) / precio_inicial) * 100
    return roi

df_roi = df_filtrado.groupby('Simbolo').apply(calcular_roi).reset_index(name='ROI (%)')

df_roi = df_roi.sort_values(by='ROI (%)', ascending=False).reset_index(drop=True)

print("--- Criptomonedas más rentables (ROI 2021-2024) ---")
print(df_roi)
print("\n")


df_volatilidad = df_filtrado.groupby('Simbolo')['Close'].agg(['mean', 'std']).reset_index()

df_volatilidad['CV (%)'] = (df_volatilidad['std'] / df_volatilidad['mean']) * 100

df_volatilidad = df_volatilidad.sort_values(by='CV (%)', ascending=False).reset_index(drop=True)

print("--- Criptomonedas más volátiles (CV 2021-2024) ---")
print(df_volatilidad[['Simbolo', 'CV (%)']])

plt.figure(figsize=(12, 6))
sns.barplot(x='Simbolo', y='CV (%)', data=df_volatilidad, palette='viridis')
plt.title('Volatilidad de las Criptomonedas (CV %)')
plt.xlabel('Criptomoneda')
plt.ylabel('Coeficiente de Variación (%)')
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()

plt.figure(figsize=(12, 6))
sns.barplot(x='Simbolo', y='ROI (%)', data=df_roi, palette='magma')
plt.title('Rentabilidad de las Criptomonedas (ROI %)')
plt.xlabel('Criptomoneda')
plt.ylabel('Retorno de Inversión (%)')
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()

df_analisis = pd.merge(df_roi, df_volatilidad[['Simbolo', 'CV (%)']], on='Simbolo')
correlacion = df_analisis['ROI (%)'].corr(df_analisis['CV (%)'])
print(f"Correlación entre ROI y CV: {correlacion:.2f}")

plt.figure(figsize=(10, 6))
sns.scatterplot(x='CV (%)', y='ROI (%)', data=df_analisis, hue='Simbolo', palette='tab10', s=100)
plt.title('Relación entre Volatilidad (CV %) y Rentabilidad (ROI %)')
plt.xlabel('Volatilidad (CV %)')
plt.ylabel('Rentabilidad (ROI %)')
plt.legend(title='Criptomoneda', bbox_to_anchor=(1.05, 1), loc='upper left')
plt.tight_layout()
plt.savefig('relacion_volatilidad_roi.png', bbox_inches='tight')
plt.show()

df_pivot = df_filtrado.pivot(index='Date', columns='Simbolo', values='Close')


df_retornos = df_pivot.pct_change().dropna()

delays = [0, 1, 2, 3]

resultados_correlacion = {}

for cripto in df_retornos.columns:
    if cripto == 'BTC': 
        continue
        
    correlaciones_cripto = {}
    for delay in delays:
        serie_desfasada = df_retornos[cripto].shift(-delay)
        
        corr = df_retornos['BTC'].corr(serie_desfasada)
        correlaciones_cripto[f'Delay {delay}'] = corr
        
    resultados_correlacion[cripto] = correlaciones_cripto

df_cross_corr = pd.DataFrame(resultados_correlacion).T

print("--- Correlación de Altcoins vs BTC (Delay en días) ---")
print(df_cross_corr)


plt.figure(figsize=(10, 6))
sns.heatmap(df_cross_corr, annot=True, cmap='coolwarm', center=0, fmt=".3f")
plt.title('Correlación Cruzada: Altcoins vs BTC (2021-2024)')
plt.ylabel('Criptomoneda')
plt.xlabel('Retraso (Días)')
plt.savefig('heatmap_correlacion_delay.png', bbox_inches='tight')
plt.show()


def calcular_recuperacion(df_cripto):
    df_cripto = df_cripto.sort_values('Date').reset_index(drop=True)
    
    df_cripto['Precio_7d_atras'] = df_cripto['Close'].shift(7)
    
    df_cripto['Variacion_7d'] = (df_cripto['Close'] - df_cripto['Precio_7d_atras']) / df_cripto['Precio_7d_atras']
    
    dias_crash = df_cripto[df_cripto['Variacion_7d'] <= -0.15]
    
    tiempos_recuperacion = []
    
    for index, row in dias_crash.iterrows():
        precio_objetivo = row['Precio_7d_atras']
        fecha_crash = row['Date']
        
        futuro = df_cripto[df_cripto['Date'] > fecha_crash]
        
        recuperacion = futuro[futuro['Close'] >= precio_objetivo]
        
        if not recuperacion.empty:
            fecha_recuperacion = recuperacion.iloc[0]['Date']
            dias_tardados = (fecha_recuperacion - fecha_crash).days
            tiempos_recuperacion.append(dias_tardados)
            
    if tiempos_recuperacion:
        return pd.Series({
            'Cantidad de Crashes': len(tiempos_recuperacion),
            'Mediana de Recuperacion (Días)': np.median(tiempos_recuperacion),
            'Recuperacion más rápida': np.min(tiempos_recuperacion)
        })
    else:
        return pd.Series({
            'Cantidad de Crashes': len(dias_crash),
            'Mediana de Recuperacion (Días)': np.nan,
            'Recuperacion más rápida': np.nan
        })

df_recuperacion = df_filtrado.groupby('Simbolo').apply(calcular_recuperacion).reset_index()

df_recuperacion = df_recuperacion[~df_recuperacion['Simbolo'].isin(['USDT', 'USDC'])]

df_recuperacion = df_recuperacion.sort_values(by='Mediana de Recuperacion (Días)').reset_index(drop=True)

print("--- Resiliencia: Mediana de días para recuperarse de una caída del 15% ---")
print(df_recuperacion)

df_analisis_recuperacion = pd.merge(df_recuperacion, df_volatilidad[['Simbolo', 'CV (%)']], on='Simbolo')
correlacion_recuperacion_volatilidad = df_analisis_recuperacion['Mediana de Recuperacion (Días)'].corr(df_analisis_recuperacion['CV (%)'])
print(f"Correlación entre Mediana de Recuperación y Volatilidad (CV): {correlacion_recuperacion_volatilidad:.2f}")

plt.figure(figsize=(10, 6))
sns.scatterplot(x='CV (%)', y='Mediana de Recuperacion (Días)', data=df_analisis_recuperacion, hue='Simbolo', palette='tab10', s=100)
plt.title('Relación entre Volatilidad (CV %) y Mediana de Recuperación (Días)')
plt.xlabel('Volatilidad (CV %)')
plt.ylabel('Mediana de Recuperación (Días)')
plt.legend(title='Criptomoneda', bbox_to_anchor=(1.05, 1), loc='upper left')
plt.tight_layout()
plt.savefig('relacion_volatilidad_recuperacion.png', bbox_inches='tight')
plt.show()


df_btc = df_filtrado[df_filtrado['Simbolo'] == 'BTC'].copy()

df_btc['Retorno_Diario'] = df_btc['Close'].pct_change()

umbral_caida = -0.05
dias_crash_btc = df_btc[df_btc['Retorno_Diario'] <= umbral_caida][['Date', 'Retorno_Diario', 'Close']].copy()

dias_crash_btc['Date_Previo'] = dias_crash_btc['Date'] - pd.Timedelta(days=1)

df_usdt = df_filtrado[df_filtrado['Simbolo'] == 'USDT'].copy()

df_usdt['Volumen_MA_20'] = df_usdt['Volume'].rolling(window=20).mean()


df_usdt['Anomalia_Volumen'] = (df_usdt['Volume'] / df_usdt['Volumen_MA_20']) - 1

analisis_stable = pd.merge(
    dias_crash_btc, 
    df_usdt[['Date', 'Volume', 'Volumen_MA_20', 'Anomalia_Volumen']], 
    left_on='Date_Previo', 
    right_on='Date', 
    suffixes=('_Crash', '_DiaPrevio')
)

umbral_volumen = 0.20
analisis_stable['Aviso_Volumen'] = analisis_stable['Anomalia_Volumen'] >= umbral_volumen

total_caidas = len(analisis_stable)
caidas_con_aviso = analisis_stable['Aviso_Volumen'].sum()
porcentaje_aviso = (caidas_con_aviso / total_caidas) * 100

print("--- Análisis de Volumen de USDT previo a caídas de BTC ---")
print(f"Total de caídas fuertes de BTC detectadas (>5% diario): {total_caidas}")
print(f"Veces que el volumen de USDT aumentó >20% el día anterior: {caidas_con_aviso}")
print(f"Efectividad de la señal: {porcentaje_aviso:.2f}%\n")

plt.figure(figsize=(9, 5))
sns.histplot(analisis_stable['Anomalia_Volumen'] * 100, bins=20, kde=True, color='purple')
plt.axvline(x=20, color='red', linestyle='--', label='Umbral del 20% de aumento')
plt.title('Aumento de Volumen de USDT el día PREVIO a una caída de BTC')
plt.xlabel('Aumento de Volumen vs Media de 20 días (%)')
plt.ylabel('Cantidad de Días (Frecuencia)')
plt.gca().xaxis.set_major_formatter(mtick.PercentFormatter())
plt.legend()

plt.savefig('volumen_stablecoins_pre_crash.png', bbox_inches='tight')
plt.show()


df_filtrado['Year'] = df_filtrado['Date'].dt.year
df_filtrado['Month'] = df_filtrado['Date'].dt.month

def roi_mensual(grupo):
    grupo = grupo.sort_values('Date')
    precio_inicio = grupo['Close'].iloc[0]
    precio_fin = grupo['Close'].iloc[-1]
    return ((precio_fin - precio_inicio) / precio_inicio) * 100

df_meses = df_filtrado.groupby(['Simbolo', 'Year', 'Month']).apply(roi_mensual).reset_index(name='ROI_Mensual')


df_meses = df_meses[~df_meses['Simbolo'].isin(['USDT', 'USDC'])]
estacionalidad = df_meses.groupby(['Simbolo', 'Month'])['ROI_Mensual'].mean().reset_index()

df_estacionalidad_pivot = estacionalidad.pivot(index='Simbolo', columns='Month', values='ROI_Mensual')

meses_nombres = ['Ene', 'Feb', 'Mar', 'Abr', 'May', 'Jun', 'Jul', 'Ago', 'Sep', 'Oct', 'Nov', 'Dic']
df_estacionalidad_pivot.columns = meses_nombres[:len(df_estacionalidad_pivot.columns)]

print("--- Rendimiento Promedio Mensual (%) [2021-2024] ---")
print(df_estacionalidad_pivot.round(2))

plt.figure(figsize=(12, 6))
sns.heatmap(df_estacionalidad_pivot, annot=True, cmap='RdYlGn', center=0, fmt=".1f", linewidths=.5)
plt.title('Estacionalidad Cripto: Rendimiento Promedio Mensual (2021-2024)')
plt.xlabel('Mes del Año')
plt.ylabel('Criptomoneda')
plt.savefig('heatmap_estacionalidad.png', bbox_inches='tight')
plt.show()

df_btc = df_filtrado[df_filtrado['Simbolo'] == 'BTC'].copy()

df_btc['Volatilidad_Intradiaria (%)'] = ((df_btc['High'] - df_btc['Low']) / df_btc['Open']) * 100

x = df_btc['Volume']
y = df_btc['Volatilidad_Intradiaria (%)']

correlacion = x.corr(y)

res = linregress(x, y)
r_cuadrado = res.rvalue**2

print("--- Relación entre Volumen y Volatilidad Intradiaria (BTC) ---")
print(f"Coeficiente de correlación (r): {correlacion:.4f}")
print(f"Coeficiente de determinación (R^2): {r_cuadrado:.4f}")
print(f"Función de regresión: y = {res.slope:.2e} * x + {res.intercept:.2f}\n")

plt.figure(figsize=(10, 6))
sns.regplot(
    x=x, 
    y=y, 
    scatter_kws={'alpha': 0.5, 'color': '#1f77b4'}, 
    line_kws={'color': 'red', 'label': f'Recta de Regresión\n$R^2$ = {r_cuadrado:.4f}'}
)
plt.title('Regresión Lineal: Volumen vs Volatilidad Intradiaria (BTC 2021-2024)')
plt.xlabel('Volumen Diario de Transacciones (USD)')
plt.ylabel('Volatilidad Intradiaria (%)')
plt.legend()
plt.savefig('regresion_volumen_volatilidad.png', bbox_inches='tight')
plt.show()


plt.figure(figsize=(14, 7))
for simbolo in df_filtrado['Simbolo'].unique():
    df_cripto = df_filtrado[df_filtrado['Simbolo'] == simbolo]
    plt.plot(df_cripto['Date'], df_cripto['Close'], label=simbolo)
plt.title('Evolución de los Precios de las Criptomonedas (2021-2024)')
plt.xlabel('Fecha')
plt.ylabel('Precio de Cierre (USD)')
plt.legend()
plt.savefig('evolucion_precios.png', bbox_inches='tight')
plt.show()

df_btc = df_filtrado[df_filtrado['Simbolo'] == 'BTC'].copy()
plt.figure(figsize=(12, 6))
plt.plot(df_btc['Date'], df_btc['Close'], color='orange')
plt.title('Evolución del Precio de Bitcoin (2021-2024)')
plt.xlabel('Fecha')
plt.ylabel('Precio de Cierre (USD)')
plt.savefig('evolucion_btc.png', bbox_inches='tight')
plt.show()

df_usdt = df_filtrado[df_filtrado['Simbolo'] == 'USDT'].copy()
plt.figure(figsize=(12, 6))
plt.plot(df_usdt['Date'], df_usdt['Close'], color='green')
plt.title('Evolución del Precio de USDT (2021-2024)')
plt.xlabel('Fecha')
plt.ylabel('Precio de Cierre (USD)')
plt.savefig('evolucion_usdt.png', bbox_inches='tight')
plt.show()

df_eth = df_filtrado[df_filtrado['Simbolo'] == 'ETH'].copy()
plt.figure(figsize=(12, 6))
plt.plot(df_eth['Date'], df_eth['Close'], color='blue')
plt.title('Evolución del Precio de Ethereum (2021-2024)')
plt.xlabel('Fecha')
plt.ylabel('Precio de Cierre (USD)')
plt.savefig('evolucion_eth.png', bbox_inches='tight')
plt.show()

df_comparacion = df_filtrado.groupby('Simbolo').agg({'Date': ['min', 'max'], 'Close': ['first', 'last']}).reset_index()
df_comparacion.columns = ['Simbolo', 'Fecha_Inicio', 'Fecha_Fin', 'Precio_Inicio', 'Precio_Fin']
df_comparacion['ROI (%)'] = ((df_comparacion['Precio_Fin'] - df_comparacion['Precio_Inicio']) / df_comparacion['Precio_Inicio']) * 100
print("--- Comparación de Precios al Inicio y Fin del Período ---")
print(df_comparacion[['Simbolo', 'Precio_Inicio', 'Precio_Fin']])
