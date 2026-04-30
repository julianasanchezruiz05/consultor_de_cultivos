import streamlit as st
import pandas as pd
import plotly.express as px

# Configuración de la página
st.set_page_config(
    page_title="Consulta de Desempeño Agrícola",
    page_icon="🌱",
    layout="wide"
)

def main():
    st.title("🌱 Consulta de Desempeño Agrícola (2019 - 2024)")
    st.markdown("""
    Esta aplicación permite analizar datos de producción agrícola en Colombia. 
    Sube tu archivo Excel para comenzar la exploración.
    """)

    # Barra lateral para carga de datos
    st.sidebar.header("Configuración de Datos")
    archivo_subido = st.sidebar.file_uploader("Carga tu archivo Excel", type=["xlsx"])
    
    # Nombre de archivo por defecto si no se sube uno
    archivo_defecto = '20250617_BaseAgricola20192024.xlsx'

    if archivo_subido is not None:
        df = cargar_datos(archivo_subido)
    else:
        try:
            df = pd.read_excel(archivo_defecto)
            st.info(f"Usando archivo local: {archivo_defecto}")
        except Exception:
            st.warning("⚠️ Por favor, sube un archivo Excel para continuar.")
            st.stop()

    # Limpieza básica de nombres de columnas
    df.columns = [col.strip() for col in df.columns]

    # --- FILTROS ---
    st.sidebar.subheader("Filtros de Búsqueda")
    
    # Selector de Departamento
    deptos = sorted(df['Departamento'].unique().tolist())
    depto_seleccionado = st.sidebar.selectbox("Selecciona el Departamento", ["Todos"] + deptos)

    # Filtrar cultivos disponibles según depto
    if depto_seleccionado != "Todos":
        df_temp = df[df['Departamento'] == depto_seleccionado]
    else:
        df_temp = df

    cultivos = sorted(df_temp['Cultivo'].unique().tolist())
    cultivo_seleccionado = st.sidebar.selectbox("Selecciona el Cultivo", ["Todos"] + cultivos)

    # Aplicar filtros al DataFrame principal
    df_filtrado = df.copy()
    if depto_seleccionado != "Todos":
        df_filtrado = df_filtrado[df_filtrado['Departamento'] == depto_seleccionado]
    if cultivo_seleccionado != "Todos":
        df_filtrado = df_filtrado[df_filtrado['Cultivo'] == cultivo_seleccionado]

    if df_filtrado.empty:
        st.error("No se encontraron registros con los filtros seleccionados.")
        return

    # Preparar métricas numéricas
    columnas_metricas = ['Área sembrada (ha)', 'Área cosechada (ha)', 'Producción (t)']
    for col in columnas_metricas:
        if col in df_filtrado.columns:
            df_filtrado[col] = pd.to_numeric(df_filtrado[col], errors='coerce')

    # --- DASHBOARD ---
    
    # KPIs Superiores
    total_prod = df_filtrado['Producción (t)'].sum()
    total_area = df_filtrado['Área sembrada (ha)'].sum()
    
    col1, col2, col3 = st.columns(3)
    col1.metric("Producción Total (t)", f"{total_prod:,.2f}")
    col2.metric("Área Sembrada Total (ha)", f"{total_area:,.2f}")
    col3.metric("Registros Encontrados", len(df_filtrado))

    # Gráficos y Tablas
    tab1, tab2, tab3 = st.tabs(["📊 Análisis Anual", "📍 Detalle Municipal", "💾 Exportar Datos"])

    with tab1:
        st.subheader(f"Evolución: {cultivo_seleccionado} en {depto_seleccionado}")
        if 'Año' in df_filtrado.columns:
            resumen_anual = df_filtrado.groupby('Año')[columnas_metricas].sum().reset_index()
            
            # Gráfico de barras interactivo
            fig = px.bar(resumen_anual, x='Año', y='Producción (t)', 
                         title="Producción por Año",
                         color_discrete_sequence=['#2E7D32'])
            st.plotly_chart(fig, use_container_width=True)
            
            st.write("Datos del Resumen Anual:")
            st.dataframe(resumen_anual, use_container_width=True)
        else:
            st.warning("La columna 'Año' no está presente en los datos.")

    with tab2:
        st.subheader("Desglose a Nivel Municipal")
        columnas_visibles = ['Año', 'Municipio', 'Cultivo', 'Área sembrada (ha)', 
                             'Área cosechada (ha)', 'Producción (t)']
        if 'Rendimiento (t/ha)' in df_filtrado.columns:
            columnas_visibles.append('Rendimiento (t/ha)')
            
        # Filtramos columnas que existan
        cols_final = [c for c in columnas_visibles if c in df_filtrado.columns]
        st.dataframe(df_filtrado[cols_final], use_container_width=True)

    with tab3:
        st.subheader("Descargar Resultados")
        csv = df_filtrado.to_csv(index=False).encode('utf-8')
        st.download_button(
            label="Descargar datos filtrados (CSV)",
            data=csv,
            file_name=f"consulta_agricola_{depto_seleccionado}_{cultivo_seleccionado}.csv",
            mime="text/csv",
        )

@st.cache_data
def cargar_datos(file):
    """Carga y cachea los datos para mejorar el rendimiento"""
    return pd.read_excel(file)

if __name__ == "__main__":
    main()