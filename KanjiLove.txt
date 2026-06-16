import streamlit as st
import pandas as pd
import random
import os
import base64

# ==========================================
# 1. CONFIGURACIÓN INICIAL
# ==========================================
ARCHIVO_ICONO = "icono_kanji.ico"
ARCHIVO_FONDO = "pngtree-asian-style-black-texture-banner-brush-stroke-object-with-japanese-wave-image_13901042.png"
ARCHIVO_EXCEL = "Copia de Marugoto Kanjis.xlsx"

st.set_page_config(
    page_title="KanjiLove ❤️",
    page_icon=ARCHIVO_ICONO if os.path.exists(ARCHIVO_ICONO) else "❤️",
    layout="centered"
)

# ==========================================
# 2. DISEÑO Y ESTILOS (CSS)
# ==========================================
def aplicar_diseno_visual():
    try:
        if os.path.exists(ARCHIVO_FONDO):
            with open(ARCHIVO_FONDO, "rb") as f:
                data = f.read()
            b64_bg = base64.b64encode(data).decode()
            fondo_css = f'background-image: url("data:image/png;base64,{b64_bg}");'
        else:
            fondo_css = 'background-color: #1a1a1a;' # Respaldo si no hay imagen

        estilos = f"""
        <style>
        .stApp {{
            {fondo_css}
            background-size: cover;
            background-position: center;
            background-attachment: fixed;
        }}
        .stMainBlockContainer {{
            background-color: rgba(255, 255, 255, 0.95) !important;
            padding: 40px !important;
            border-radius: 40px !important;
            box-shadow: 0px 15px 40px rgba(0,0,0,0.6) !important;
            border: 3px solid #E74C3C !important;
            margin-top: 30px !important;
        }}
        h1, h2, h3, p, label {{
            color: #1A1A1A !important;
        }}
        
        /* --- AQUÍ ESTÁ LA CORRECCIÓN DE LOS BOTONES --- */
        .stButton > button {{
            border-radius: 30px !important;
            border: 2px solid #E74C3C !important;
            background-color: #2C3E50 !important; /* El fondo azul oscuro */
            color: #FFFFFF !important; /* Blanco puro */
            font-weight: bold !important;
            padding: 10px 20px !important;
            transition: all 0.3s ease;
        }}
        
        /* Obligamos a que cualquier texto interno del botón también sea blanco */
        .stButton > button p, .stButton > button div {{
            color: #FFFFFF !important; 
        }}
        
        .stButton > button:hover {{
            background-color: #E74C3C !important;
            border-color: #2C3E50 !important;
        }}
        
        /* Aseguramos que al pasar el ratón siga siendo blanco */
        .stButton > button:hover p, .stButton > button:hover div {{
            color: #FFFFFF !important;
        }}
        </style>
        """
        st.markdown(estilos, unsafe_allow_html=True)
    except Exception as e:
        st.error(f"Error al cargar el diseño visual: {e}")

aplicar_diseno_visual()

# ==========================================
# 3. ESTADOS DE LA APLICACIÓN (SESSION STATE)
# ==========================================
if 'pantalla' not in st.session_state: st.session_state.pantalla = 'menu_modos'
if 'modo_juego' not in st.session_state: st.session_state.modo_juego = None
if 'aciertos' not in st.session_state: st.session_state.aciertos = 0
if 'intentos' not in st.session_state: st.session_state.intentos = 0
if 'pendientes' not in st.session_state: st.session_state.pendientes = []
if 'kanjis' not in st.session_state: st.session_state.kanjis = []
if 'espanol' not in st.session_state: st.session_state.espanol = []
if 'lecturas' not in st.session_state: st.session_state.lecturas = []
if 'respondido' not in st.session_state: st.session_state.respondido = False

# ==========================================
# 4. FUNCIONES LÓGICAS
# ==========================================
def obtener_lectura(fila):
    kun = str(fila.get('訓「読み」', '')).strip()
    on = str(fila.get('音「読み」', '')).strip()
    kun = "" if kun in ["nan", "None", "-", ""] else kun
    on = "" if on in ["nan", "None", "-", ""] else on
    if kun and on: return f"{kun} / {on}"
    elif kun: return kun
    elif on: return on
    else: return "読みなし"

def preparar_siguiente_tarjeta():
    if not st.session_state.pendientes:
        st.session_state.pantalla = 'finalizado'
        return
    
    st.session_state.indice_actual = st.session_state.pendientes.pop(0)
    st.session_state.kanji_actual = st.session_state.kanjis[st.session_state.indice_actual]
    
    if st.session_state.modo_juego == "traduccion":
        st.session_state.respuesta_correcta = st.session_state.espanol[st.session_state.indice_actual]
        lista_global = st.session_state.espanol
    else:
        st.session_state.respuesta_correcta = st.session_state.lecturas[st.session_state.indice_actual]
        lista_global = st.session_state.lecturas
        
    opciones = [st.session_state.respuesta_correcta]
    while len(opciones) < 4:
        falsa = random.choice(lista_global)
        if falsa not in opciones and pd.notna(falsa) and str(falsa).strip() != "":
            opciones.append(falsa)
            
    random.shuffle(opciones)
    st.session_state.opciones = opciones
    st.session_state.respondido = False

def iniciar_nivel(nombre_hoja):
    if not os.path.exists(ARCHIVO_EXCEL):
        st.error(f"No se encontró el archivo '{ARCHIVO_EXCEL}' en la misma carpeta que el código.")
        return
        
    try:
        df = pd.read_excel(ARCHIVO_EXCEL, sheet_name=nombre_hoja)
        df = df.dropna(subset=['漢字'])
        
        st.session_state.kanjis = df['漢字'].tolist()
        st.session_state.espanol = df['スペイン語'].tolist()
        st.session_state.lecturas = [obtener_lectura(r) for _, r in df.iterrows()]
        
        st.session_state.pendientes = list(range(len(st.session_state.kanjis)))
        random.shuffle(st.session_state.pendientes)
        
        st.session_state.aciertos = 0
        st.session_state.intentos = 0
        
        preparar_siguiente_tarjeta()
        st.session_state.pantalla = 'juego'
        
    except Exception as e:
        st.error(f"Error al leer la hoja '{nombre_hoja}': {e}")

# ==========================================
# 5. RENDERIZADO DE PANTALLAS (INTERFAZ)
# ==========================================

if st.session_state.pantalla == 'menu_modos':
    st.markdown("<h1 style='text-align: center; color: #E74C3C !important;'>KanjiLove ❤️</h1>", unsafe_allow_html=True)
    st.markdown("<p style='text-align: center; font-style: italic; font-size: 18px;'>Selecciona el Modo de Juego</p>", unsafe_allow_html=True)
    st.write("")
    
    if st.button("Traducción 🇪🇸 (Significado)", use_container_width=True):
        st.session_state.modo_juego = "traduccion"
        st.session_state.pantalla = 'menu_niveles'
        st.rerun()
        
    if st.button("Lectura 🇯🇵 (Pronunciación)", use_container_width=True):
        st.session_state.modo_juego = "lectura"
        st.session_state.pantalla = 'menu_niveles'
        st.rerun()

elif st.session_state.pantalla == 'menu_niveles':
    if st.button("← Cambiar Modo"):
        st.session_state.pantalla = 'menu_modos'
        st.rerun()
        
    st.markdown("<h2 style='text-align: center;'>Selecciona tu Nivel</h2>", unsafe_allow_html=True)
    modo_txt = "Traducción" if st.session_state.modo_juego == "traduccion" else "Lectura"
    st.markdown(f"<p style='text-align: center; font-weight: bold; color: #E74C3C !important;'>Modo: {modo_txt}</p>", unsafe_allow_html=True)
    st.write("")
    
    for nivel in ["Nivel A1", "Nivel A2.1", "Nivel A2.2"]:
        if st.button(nivel, use_container_width=True):
            iniciar_nivel(nivel)
            st.rerun()

elif st.session_state.pantalla == 'juego':
    col_back, col_score = st.columns([1, 1])
    with col_back:
        if st.button("← Menú Principal"):
            st.session_state.pantalla = 'menu_niveles'
            st.rerun()
    with col_score:
        st.markdown(f"<p style='text-align: right; font-weight: bold; font-size: 18px;'>Puntuación: <span style='color: #E74C3C;'>{st.session_state.aciertos} / {st.session_state.intentos}</span></p>", unsafe_allow_html=True)
        
    st.markdown("<hr style='border: 1px solid #E0E0E0; margin: 10px 0;'>", unsafe_allow_html=True)
    st.markdown(f"<h1 style='text-align: center; font-size: 90px; margin: 20px 0;'>{st.session_state.kanji_actual}</h1>", unsafe_allow_html=True)
    
    if not st.session_state.respondido:
        for opcion in st.session_state.opciones:
            if st.button(opcion, use_container_width=True):
                st.session_state.intentos += 1
                st.session_state.respondido = True
                
                if opcion == st.session_state.respuesta_correcta:
                    st.session_state.aciertos += 1
                    st.session_state.es_correcto = True
                else:
                    st.session_state.es_correcto = False
                st.rerun()
                
    else:
        lectura_info = st.session_state.lecturas[st.session_state.indice_actual]
        significado_info = st.session_state.espanol[st.session_state.indice_actual]
        
        if st.session_state.es_correcto:
            st.markdown("""
            <div style="background-color: #E8F8F5; color: #117A65; padding: 20px; border-radius: 30px; text-align: center; font-size: 18px; font-weight: bold; border: 2px solid #1ABC9C; margin-bottom: 20px;">
                ¡Correcto! ✨ Manuel-san estaría orgulloso.
            </div>
            """, unsafe_allow_html=True)
        else:
            st.markdown(f"""
            <div style="background-color: #FDEDEC; color: #C0392B; padding: 20px; border-radius: 30px; text-align: center; font-size: 18px; font-weight: bold; border: 2px solid #E74C3C; margin-bottom: 20px;">
                Incorrecto ❌. No sé si Manuel-san podrá perdonarte... <br>
                <span style="font-size: 15px; font-weight: normal; color: #333;">(La respuesta era: <b>{st.session_state.respuesta_correcta}</b>)</span>
            </div>
            """, unsafe_allow_html=True)
            
        st.markdown(f"""
        <div style="background-color: #FDFEFE; padding: 20px; border-radius: 25px; border-left: 8px solid #E74C3C; box-shadow: 0px 4px 10px rgba(0,0,0,0.05); margin-bottom: 20px;">
            <p style="margin: 0; font-weight: bold; font-size: 16px; color: #E74C3C;">📝 Repaso rápido:</p>
            <p style="margin: 8px 0 0 0;"><b>Kanji:</b> <span style="font-size: 20px;">{st.session_state.kanji_actual}</span></p>
            <p style="margin: 4px 0 0 0;"><b>Lectura:</b> {lectura_info}</p>
            <p style="margin: 4px 0 0 0;"><b>Significado:</b> {significado_info}</p>
        </div>
        """, unsafe_allow_html=True)
        
        marcar_dificil = st.checkbox("Marcar como difícil (Repetir pronto) 🔁")
        
        if st.button("Siguiente Tarjeta ➡️", use_container_width=True):
            if not st.session_state.es_correcto or marcar_dificil:
                pos = min(len(st.session_state.pendientes), random.randint(2, 4))
                st.session_state.pendientes.insert(pos, st.session_state.indice_actual)
                
            preparar_siguiente_tarjeta()
            st.rerun()

elif st.session_state.pantalla == 'finalizado':
    st.balloons()
    st.markdown("<h2 style='text-align: center; color: #E74C3C !important;'>¡Nivel Completado! 🏆</h2>", unsafe_allow_html=True)
    st.markdown("<p style='text-align: center; font-size: 18px;'>¡Felicidades! Has terminado todas las tarjetas.</p>", unsafe_allow_html=True)
    st.markdown(f"""
    <div style="background-color: #F8F9F9; padding: 20px; border-radius: 30px; text-align: center; margin: 20px 0; border: 2px solid #E74C3C;">
        <h3 style='margin: 0;'>Puntuación final: <span style='color: #E74C3C;'>{st.session_state.aciertos} / {st.session_state.intentos}</span></h3>
    </div>
    """, unsafe_allow_html=True)
    
    if st.button("Volver al Menú", use_container_width=True):
        st.session_state.pantalla = 'menu_niveles'
        st.rerun()