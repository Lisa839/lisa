import streamlit as st
import openai
import replicate
import json
import os
from datetime import datetime
from pathlib import Path

# --- Load secrets (API keys) ---
openai.api_key = st.secrets["OPENAI_API_KEY"]
replicate.Client(api_token=st.secrets["REPLICATE_API_TOKEN"])

# --- Setup file paths ---
USERS_FILE = "users.json"
LYRICS_DIR = Path("lyrics")
LYRICS_DIR.mkdir(exist_ok=True)

# --- Load/save user data ---
def load_users():
    if os.path.exists(USERS_FILE):
        with open(USERS_FILE, "r") as f:
            return json.load(f)
    return {}

def save_users(users):
    with open(USERS_FILE, "w") as f:
        json.dump(users, f, indent=2)

# --- Auth UI ---
def auth_ui():
    st.sidebar.title("Login / Signup")
    users = load_users()
    mode = st.sidebar.radio("Choose mode", ["Login", "Signup"])
    username = st.sidebar.text_input("Username")
    password = st.sidebar.text_input("Password", type="password")

    if st.sidebar.button("Continue"):
        if mode == "Signup":
            if username in users:
                st.sidebar.error("Username already exists.")
            else:
                users[username] = {"password": password, "lyrics": []}
                save_users(users)
                st.session_state["user"] = username
                st.sidebar.success("Account created!")
        else:
            if username in users and users[username]["password"] == password:
                st.session_state["user"] = username
                st.sidebar.success("Logged in!")
            else:
                st.sidebar.error("Invalid credentials.")

# --- Generate lyrics ---
def generate_lyrics(theme, style, mood):
    prompt = f"Write a {style} song about {theme}. Make it {mood}.\n\nLyrics:\n"
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "system", "content": "You write song lyrics."},
                  {"role": "user", "content": prompt}],
        temperature=0.9,
        max_tokens=400
    )
    return response["choices"][0]["message"]["content"]

# --- Generate music ---
def generate_music(prompt_text):
    output = replicate.run(
        "facebook/musicgen-medium",
        input={"prompt": prompt_text}
    )
    return output

# --- Save lyrics to user ---
def save_lyrics(user, title, lyrics):
    users = load_users()
    file_path = LYRICS_DIR / f"{title}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.txt"
    with open(file_path, "w") as f:
        f.write(lyrics)
    users[user]["lyrics"].append({
        "title": title,
        "file": str(file_path),
        "date": datetime.now().isoformat()
    })
    save_users(users)

# --- Main ---
if "user" not in st.session_state:
    auth_ui()
else:
    st.sidebar.success(f"Logged in as {st.session_state['user']}")
    if st.sidebar.button("Logout"):
        del st.session_state["user"]
        st.experimental_rerun()

    st.title("🎵 Lyric Writing AI")

    theme = st.text_input("Theme", "falling in love")
    style = st.selectbox("Genre", ["Pop", "Rap", "Rock", "Country", "Lo-fi"])
    mood = st.text_input("Mood", "uplifting")

    if st.button("Generate Lyrics"):
        with st.spinner("Writing lyrics..."):
            lyrics = generate_lyrics(theme, style, mood)
            st.session_state["lyrics"] = lyrics

    if "lyrics" in st.session_state:
        st.subheader("Your Lyrics")
        edited = st.text_area("Edit", st.session_state["lyrics"], height=300)
        title = st.text_input("Song Title", "My Song")

        if st.button("Save Lyrics"):
            save_lyrics(st.session_state["user"], title, edited)
            st.success("Lyrics saved!")

        if st.button("Generate Background Music"):
            with st.spinner("Composing..."):
                urls = generate_music(f"{style} instrumental, {mood}")
                if urls:
                    st.audio(urls[0], format="audio/mp3")

    st.subheader("📁 Saved Lyrics")
    user_data = load_users().get(st.session_state["user"], {})
    for entry in reversed(user_data.get("lyrics", [])):
        st.markdown(f"**{entry['title']}** ({entry['date'][:10]})")
        with open(entry["file"]) as f:
            st.code(f.read())
