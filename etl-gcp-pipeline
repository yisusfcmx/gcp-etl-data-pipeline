import pandas as pd
import io
import json
import re
import csv
from decimal import Decimal, ROUND_HALF_UP
from google.colab import files
from tqdm.notebook import tqdm
from google.cloud import bigquery

PROYECTO_ID = "" # @param {type:"string"}
TABLA_DESTINO_INPUT = "" # @param {type:"string"}

def limpiar_monto(valor):
    if pd.isna(valor) or str(valor).strip() in ["", "-", "NULL"]: return Decimal('0.00')
    try:
        clean = re.sub(r'[^0-9.\-]', '', str(valor))
        return Decimal(clean)
    except: return Decimal('0.00')

def limpieza_geografica_pro(valor):
    if pd.isna(valor): return ""
    texto = str(valor).replace('\xa0', ' ')
    return " ".join(texto.split()).upper().strip()

print("Selecciona el JSON Maestro...")
up_json = files.upload()
manifiesto = json.loads(up_json[list(up_json.keys())[0]].decode('utf-8'))

TABLA_JSON = manifiesto.get('configuracion', {}).get('tabla_id')
if TABLA_DESTINO_INPUT != TABLA_JSON:
    print(f"ERROR: La tabla ingresada no coincide con el JSON Maestro.")
    print(f"Ingresado: {TABLA_DESTINO_INPUT}\nEn JSON: {TABLA_JSON}")
    raise Exception("Validacion de seguridad fallida.")
print(f"Tabla validada: {TABLA_JSON}")

print("\nSelecciona el CSV de Datos...")
up_csv = files.upload()
csv_name = list(up_csv.keys())[0]

try:
    content = up_csv[csv_name].decode('utf-8')
except:
    content = up_csv[csv_name].decode('latin-1')

sep = csv.Sniffer().sniff(content[:20000], delimiters='|,;').delimiter
df_sucio = pd.read_csv(io.StringIO(content), sep=sep, dtype=str, low_memory=False)

esquema = manifiesto.get('esquema', [])
if len(esquema) != len(df_sucio.columns):
    raise Exception(f"Error Estructural: JSON ({len(esquema)} col) != CSV ({len(df_sucio.columns)} col)")

df_final = pd.DataFrame()
print(f"Procesando {len(df_sucio):,} registros...")

for n in tqdm(range(len(esquema)), desc="Limpiando Datos"):
    col = esquema[n]
    nombre, tipo = col['name'], col['type']
    serie = df_sucio.iloc[:, n]

    if tipo == 'STRING':
        df_final[nombre] = serie.apply(limpieza_geografica_pro)
    elif tipo == 'NUMERIC':
        df_final[nombre] = serie.apply(limpiar_monto)
    elif tipo == 'DATE':
        temp_dates = pd.to_datetime(serie, dayfirst=True, errors='coerce')
        nulos = temp_dates.isna().sum()
        if nulos > 0:
            print(f"Alerta: {nulos} fechas en '{nombre}' seran NULL.")
        df_final[nombre] = temp_dates.dt.strftime('%Y-%m-%d')
    elif tipo == 'INTEGER':
        df_final[nombre] = pd.to_numeric(serie.str.replace(r'[^0-9\-]', '', regex=True), errors='coerce').fillna(0).astype(int)
    else:
        df_final[nombre] = serie.str.strip().fillna('')

print("\n" + "="*50)
print(f"PROCESAMIENTO COMPLETADO")
print(f"Destino: {TABLA_JSON}")
print(f"Filas: {len(df_final):,}")

display(df_final.head())

print("\nIniciando carga a BigQuery...")
try:
    import pandas_gbq

    pandas_gbq.to_gbq(
        df_final,
        destination_table=TABLA_JSON,
        project_id=PROYECTO_ID,
        if_exists='append'
    )
    print(f"Carga exitosa: {len(df_final)} registros cargados.")
except Exception as e:
    print(f"Error en la carga: {e}")
