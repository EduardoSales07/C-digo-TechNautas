# ================================================================
#  TechNautas Bot Unificado (Gemini) - 5 notícias + Legenda + Vídeo
# ================================================================

import os
import json
import requests
import feedparser
from datetime import datetime
from dotenv import load_dotenv
import google.generativeai as genai

# ================================================================
# 1️⃣ CARREGAR .ENV
# ================================================================
load_dotenv()
API_KEY_GEMINI = os.getenv("GOOGLE_API_KEY")
URL_NOTICIAS = os.getenv("URL_NOTICIAS")  # Opcional

if not API_KEY_GEMINI:
    print("❌ Erro: GOOGLE_API_KEY não encontrada no .env")
    exit()

genai.configure(api_key=API_KEY_GEMINI)
modelo = genai.GenerativeModel("models/gemini-2.5-flash")

ARQUIVO_JSON = "posts_gerados.json"

RSS_FEEDS = [
    "https://www.theverge.com/rss/index.xml",
    "https://techcrunch.com/feed/",
    "https://www.engadget.com/rss.xml"
]

# ================================================================
# 2️⃣ JSON
# ================================================================
def carregar_posts():
    try:
        with open(ARQUIVO_JSON, "r", encoding="utf-8") as f:
            return json.load(f)
    except:
        return []

def salvar_posts(lista):
    with open(ARQUIVO_JSON, "w", encoding="utf-8") as f:
        json.dump(lista, f, indent=4, ensure_ascii=False)
    print(f"\n💾 {len(lista)} posts salvos em {ARQUIVO_JSON}!")

# ================================================================
# 3️⃣ NORMALIZAÇÃO
# ================================================================
def normalizar_lista(raw_list):
    noticias = []
    for item in raw_list:
        if not isinstance(item, dict):
            continue
        noticias.append({
            "title": item.get("title") or "Sem título",
            "description": item.get("description") or "",
            "url": item.get("url") or ""
        })
    return noticias

# ================================================================
# 4️⃣ BUSCAR 5 NOTÍCIAS
# ================================================================
def buscar_noticias():
    noticias_final = []

    # API primeiro
    if URL_NOTICIAS:
        print("🔍 Tentando buscar via API...")
        try:
            r = requests.get(URL_NOTICIAS, timeout=10)
            if r.status_code == 200:
                dados = r.json()
                artigos = dados.get("articles") or dados.get("results") or []
                artigos = artigos[:5]
                noticias = normalizar_lista(artigos)
                if noticias:
                    print(f"📌 {len(noticias)} notícias via API.")
                    return noticias
        except Exception as e:
            print("⚠️ Erro na API:", e)

    # RSS fallback
    print("🔁 Usando RSS...")
    for feed_url in RSS_FEEDS:
        feed = feedparser.parse(feed_url)
        for entry in feed.entries:
            noticias_final.append({
                "title": entry.get("title", "Sem título"),
                "description": entry.get("summary", ""),
                "url": entry.get("link", "")
            })
            if len(noticias_final) >= 5:
                break
        if len(noticias_final) >= 5:
            break

    print(f"📌 Total coletado: {len(noticias_final)}")
    return noticias_final

# ================================================================
# 5️⃣ RESUMO
# ================================================================
def resumir_noticia(noticia):
    prompt = f"""
    Traduza e resuma a seguinte notícia para o português.
    Máximo: 3 frases curtas e modernas.
    Estilo: TechNautas.

    Título: {noticia['title']}
    Conteúdo: {noticia['description']}
    """
    try:
        r = modelo.generate_content(prompt)
        return r.text.strip()
    except Exception as e:
        return f"Erro ao resumir: {e}"

# ================================================================
# 6️⃣ LEGENDA DE INSTAGRAM
# ================================================================
def gerar_legenda(noticia):
    prompt = f"""
    Gere uma legenda para Instagram baseada nesta notícia.
    Estilo: TechNautas, moderno, direto e chamativo.
    Inclua CTA.
    Inclua 10 hashtags.

    Título: {noticia['title']}
    Descrição: {noticia['description']}
    """
    try:
        r = modelo.generate_content(prompt)
        return r.text.strip()
    except:
        return "Erro ao gerar legenda."

# ================================================================
# 7️⃣ PROMPT PARA GERAR VÍDEO (IA)
# ================================================================
def gerar_prompt_video(noticia):
    prompt = f"""
    Gere um prompt cinematográfico para criar um vídeo curto (15 a 25s)
    sobre esta notícia.

    O prompt deve conter:
    - cenário moderno e tecnológico
    - transições suaves
    - elementos visuais que representem o tema
    - clima futurista
    - instruções claras para IA de vídeo

    Notícia:
    {noticia['title']}
    {noticia['description']}
    """
    try:
        r = modelo.generate_content(prompt)
        return r.text.strip()
    except:
        return "Erro ao gerar prompt de vídeo."

# ================================================================
# 8️⃣ ROTINA PRINCIPAL
# ================================================================
def main():
    print("\n🚀 TechNautas Bot iniciado...\n")

    posts_existentes = carregar_posts()
    noticias = buscar_noticias()

    novos_posts = []

    for noticia in noticias:
        print(f"\n📰 {noticia['title']}")

        resumo = resumir_noticia(noticia)
        legenda = gerar_legenda(noticia)
        prompt_video = gerar_prompt_video(noticia)

        novo = {
            "titulo": noticia["title"],
            "resumo": resumo,
            "legenda_instagram": legenda,
            "prompt_video": prompt_video,
            "url": noticia["url"],
            "data": datetime.now().isoformat()
        }

        novos_posts.append(novo)

    if novos_posts:
        posts_existentes.extend(novos_posts)
        salvar_posts(posts_existentes)

# ================================================================
if __name__ == "__main__":
    main()
