import streamlit as st
from collections import defaultdict
import re
import pandas as pd
import matplotlib.pyplot as plt

# ------------------- Elo-Faktor -------------------
K = 50

# ------------------- Spieler-Statistiken -------------------
players = defaultdict(lambda: {
    "elo_all": 1000,
    "elo_1v1": 1000,
    "elo_2v2": 1000,
    "wins": 0,
    "losses": 0,
    "games": 0,
    "games_1v1": 0,
    "games_2v2": 0,
    "goals_for": 0,
    "goals_against": 0,
    "streak": 0,
    "elo_history_all": [1000],
    "elo_history_1v1": [1000],
    "elo_history_2v2": [1000],
})

history = []

# ------------------- Funktionen -------------------
def parse_line(line):
    line = line.strip()
    if not line:
        return None
    parts = re.split(r'\t+|\s{2,}|\s', line)
    nums = [p for p in parts if p.isdigit()]
    if len(nums) != 2:
        return None
    g1, g2 = map(int, nums)
    idx1 = parts.index(str(g1))
    idx2 = parts.index(str(g2))
    left_parts = parts[:idx1]
    right_parts = parts[idx2+1:]
    if not left_parts or not right_parts:
        return None
    left = " ".join(left_parts).replace("//","/").split("/")
    right = " ".join(right_parts).replace("//","/").split("/")
    return left, g1, g2, right

def add_result(team, goals_for, goals_against):
    for p in team:
        players[p]["games"] += 1
        players[p]["goals_for"] += goals_for
        players[p]["goals_against"] += goals_against

def update_elo_team(winners, losers, game_type="all"):
    key = "elo_all"
    if game_type == "1v1":
        key = "elo_1v1"
    elif game_type == "2v2":
        key = "elo_2v2"

    avg_win_elo = sum(players[p][key] for p in winners) / len(winners)
    avg_lose_elo = sum(players[p][key] for p in losers) / len(losers)
    expected = 1 / (1 + 10 ** ((avg_lose_elo - avg_win_elo) / 400))
    delta = K * (1 - expected)
    for p in winners:
        players[p][key] += delta
    for p in losers:
        players[p][key] -= delta
    return round(delta)

# ------------------- Streamlit UI -------------------
st.set_page_config(layout="wide")
st.title("Tischkicker Rangliste")

# Sidebar für Filter
st.sidebar.subheader("Filter")
elo_mode = st.sidebar.selectbox("ELO anzeigen für:", ["Alle","1v1","2v2"])

uploaded_file = st.file_uploader("Wähle eine Textdatei mit den Spielen", type=["txt"])

if uploaded_file is not None:
    for line in uploaded_file:
        line = line.decode("utf-8")
        parsed = parse_line(line)
        if parsed is None:
            continue
        team_left, g1, g2, team_right = parsed
        add_result(team_left, g1, g2)
        add_result(team_right, g2, g1)

        game_type = "1v1" if len(team_left)==1 and len(team_right)==1 else "2v2"

        if game_type=="1v1":
            for p in team_left + team_right:
                players[p]["games_1v1"] += 1
        else:
            for p in team_left + team_right:
                players[p]["games_2v2"] += 1

        winners, losers = (team_left, team_right) if g1>g2 else (team_right, team_left)
        score = f"{max(g1,g2)}:{min(g1,g2)}"

        for p in winners:
            players[p]["wins"] += 1
        for p in losers:
            players[p]["losses"] += 1

        for p in players:
            if p in winners:
                players[p]["streak"] = players[p]["streak"]+1 if players[p]["streak"]>=0 else 1
            elif p in losers:
                players[p]["streak"] = players[p]["streak"]-1 if players[p]["streak"]<=0 else -1

        delta_type = update_elo_team(winners, losers, game_type)
        delta_all = update_elo_team(winners, losers, "all")

        for p in winners + losers:
            players[p]["elo_history_all"].append(round(players[p]["elo_all"]))
            players[p]["elo_history_1v1"].append(round(players[p]["elo_1v1"]))
            players[p]["elo_history_2v2"].append(round(players[p]["elo_2v2"]))

        history.append({
            "winners": winners,
            "losers": losers,
            "score": score,
            "delta_type": delta_type,
            "delta_all": delta_all,
            "game_type": game_type
        })

    # ------------------- Tabelle erstellen -------------------
    table = []

    for name, p in players.items():
        if elo_mode=="1v1":
            games = p["games_1v1"]
            wins = sum(1 for h in history if name in h["winners"] and h["game_type"]=="1v1")
            losses = sum(1 for h in history if name in h["losers"] and h["game_type"]=="1v1")
            elo = round(p["elo_1v1"])
            elo_history = p["elo_history_1v1"]
        elif elo_mode=="2v2":
            games = p["games_2v2"]
            wins = sum(1 for h in history if name in h["winners"] and h["game_type"]=="2v2")
            losses = sum(1 for h in history if name in h["losers"] and h["game_type"]=="2v2")
            elo = round(p["elo_2v2"])
            elo_history = p["elo_history_2v2"]
        else:
            games = p["games"]
            wins = p["wins"]
            losses = p["losses"]
            elo = round(p["elo_all"])
            elo_history = p["elo_history_all"]

        if games==0:
            continue

        winrate = (wins/games*100) if games else 0
        streak = p["streak"]
        if streak>0:
            streak_display = f"🟢 +{streak}"
        elif streak<0:
            streak_display = f"🔴 {abs(streak)}"
        else:
            streak_display = "0"

        table.append({
            "Name": name,
            "ELO": elo,
            "Spiele": games,
            "Wins": wins,
            "Losses": losses,
            "Tore": f"{p['goals_for']}:{p['goals_against']}",
            "Win%": f"{winrate:.1f}%",
            "Streak": streak_display,
            "ELO_History": elo_history
        })

    df = pd.DataFrame(table)

    # ------------------- Layout mit breiten Spalten -------------------
    col1, col2 = st.columns([4,6])  # 40% / 60% Bildschirmbreite

    with col1:
        st.subheader("Match-History")
        match_rows = []
        for h in history:
            match_rows.append({
                "Gewinner": " & ".join(h["winners"]),
                "Verlierer": " & ".join(h["losers"]),
                "Score": h["score"],
                "Δ ELO Typ": h["delta_type"],
                "Δ ELO Gesamt": h["delta_all"],
                "Spieltyp": h["game_type"]
            })
        df_history = pd.DataFrame(match_rows)
        st.dataframe(df_history, use_container_width=True)

    with col2:
        st.subheader("Rangliste")
        st.dataframe(df.drop(columns="ELO_History").sort_values("ELO", ascending=False), use_container_width=True)

        st.subheader("ELO-Verlauf eines Spielers")
        selected_player = st.selectbox("Spieler auswählen", df["Name"])
        if elo_mode=="1v1":
            hist = players[selected_player]["elo_history_1v1"]
        elif elo_mode=="2v2":
            hist = players[selected_player]["elo_history_2v2"]
        else:
            hist = players[selected_player]["elo_history_all"]

        fig, ax = plt.subplots(figsize=(10,4))
        ax.plot(hist, marker="o")
        ax.set_title(f"ELO-Verlauf: {selected_player} ({elo_mode})")
        ax.set_xlabel("Spiele")
        ax.set_ylabel("ELO")
        st.pyplot(fig)
