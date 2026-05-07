import { useState, useEffect, useMemo } from "react";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_ANON_KEY
);

const ADMIN_USER = "admin";
const ADMIN_PASS = "gol2025";
const COLORS = ["#22c55e","#3b82f6","#f59e0b","#ec4899","#8b5cf6","#06b6d4","#f97316","#a3e635"];

const effColor = e => e >= 67 ? "#22c55e" : e >= 40 ? "#f59e0b" : "#ef4444";
const resColor = r => r === "V" ? "#22c55e" : r === "D" ? "#ef4444" : "#f59e0b";

function ranked(stats) {
  return [...stats].sort((a, b) => b.pts !== a.pts ? b.pts - a.pts : b.eff - a.eff);
}

function calcStats(part, partidos) {
  const pts = part.pg * 3 + part.pe;
  const pj = part.pg + part.pe + part.pp;
  const eff = pj ? Math.round((pts / (pj * 3)) * 100) : 0;
  const wr = pj ? Math.round((part.pg / pj) * 100) : 0;
  let cW = 0, cL = 0, mW = 0, mL = 0;
  const recent = [];
  const myMatches = partidos
    .filter(m => m.equipo_a.includes(part.jugador_id) || m.equipo_b.includes(part.jugador_id))
    .sort((a, b) => a.semana - b.semana);
  myMatches.forEach(m => {
    const inA = m.equipo_a.includes(part.jugador_id);
    let r;
    if (m.resultado === "draw") r = "E";
    else if ((m.resultado === "A" && inA) || (m.resultado === "B" && !inA)) r = "V";
    else r = "D";
    if (r === "V") { cW++; cL = 0; mW = Math.max(mW, cW); }
    else if (r === "D") { cL++; cW = 0; mL = Math.max(mL, cL); }
    else { cW = 0; cL = 0; }
    recent.push(r);
  });
  return { ...part, pts, pj, eff, wr, maxWin: mW, maxLoss: mL, curWin: cW, curLoss: cL, recent: recent.slice(-5) };
}

function snakeDraft(pool) {
  const s = [...pool].sort((a, b) => b.pts !== a.pts ? b.pts - a.pts : b.eff - a.eff);
  const A = [], B = [];
  s.forEach((p, i) => { (i % 4 < 2 ? A : B).push(p); });
  return { A, B };
}

const ST = {
  root: { minHeight: "100vh", background: "#080f1c", color: "#e2e8f0", fontFamily: "system-ui,sans-serif" },
  hdr: { background: "#0f172a", borderBottom: "2px solid #22c55e", padding: "11px 16px", display: "flex", alignItems: "center", justifyContent: "space-between", position: "sticky", top: 0, zIndex: 50 },
  nav: { display: "flex", borderBottom: "1px solid #1e293b", background: "#080f1c", overflowX: "auto" },
  main: { padding: "14px 16px", maxWidth: "820px", margin: "0 auto" },
  card: { background: "#1e293b", borderRadius: "12px", padding: "14px 16px", marginBottom: "10px" },
  field: { background: "#1e293b", border: "1px solid #334155", borderRadius: "8px", color: "#e2e8f0", padding: "10px 14px", fontSize: "14px", width: "100%", outline: "none", boxSizing: "border-box" },
  btnGreen: { background: "#22c55e", color: "#050d18", border: "none", borderRadius: "8px", padding: "8px 16px", fontWeight: 700, cursor: "pointer", fontSize: "13px" },
  btnGhost: { background: "transparent", border: "1px solid #334155", color: "#94a3b8", borderRadius: "8px", padding: "7px 13px", cursor: "pointer", fontSize: "13px" },
  btnDanger: { background: "transparent", border: "1px solid #7f1d1d55", color: "#f87171", borderRadius: "8px", padding: "4px 10px", cursor: "pointer", fontSize: "12px" },
  btnBlue: { background: "transparent", border: "1px solid #1e40af55", color: "#60a5fa", borderRadius: "8px", padding: "4px 10px", cursor: "pointer", fontSize: "12px" },
  tag: { display: "inline-block", borderRadius: "20px", padding: "2px 9px", fontSize: "11px", fontWeight: 700 },
  kpiGrid: { display: "grid", gridTemplateColumns: "repeat(auto-fit,minmax(140px,1fr))", gap: "9px", marginBottom: "14px" },
  kpi: { background: "#1e293b", borderRadius: "10px", padding: "12px 13px" },
  pgrid: { display: "grid", gridTemplateColumns: "repeat(auto-fill,minmax(115px,1fr))", gap: "7px", marginBottom: "12px" },
};

function Bubble({ r }) {
  return <span style={{ display: "inline-flex", alignItems: "center", justifyContent: "center", width: 19, height: 19, borderRadius: "50%", background: resColor(r) + "22", color: resColor(r), fontSize: 10, fontWeight: 800 }}>{r}</span>;
}

function KpiCard({ icon, label, val, sub, accent = "#22c55e" }) {
  return (
    <div style={{ ...ST.kpi, border: `1px solid ${accent}22` }}>
      <div style={{ fontSize: 10, color: "#64748b", letterSpacing: 2, marginBottom: 5 }}>{icon} {label}</div>
      <div style={{ fontSize: 19, fontWeight: 900, color: accent, lineHeight: 1 }}>{val}</div>
      {sub && <div style={{ fontSize: 11, color: "#475569", marginTop: 3 }}>{sub}</div>}
    </div>
  );
}

function Empty({ text }) {
  return <div style={{ textAlign: "center", color: "#334155", padding: "48px 0", fontSize: 15 }}>{text}</div>;
}

function LoginScreen({ onLogin }) {
  const [mode, setMode] = useState(null);
  const [user, setUser] = useState("");
  const [pwd, setPwd] = useState("");
  const [err, setErr] = useState("");

  function doLogin() {
    if (user.trim() === ADMIN_USER && pwd === ADMIN_PASS) onLogin("admin");
    else setErr("Usuario o contraseña incorrectos");
  }

  return (
    <div style={{ minHeight: "100vh", background: "#080f1c", display: "flex", alignItems: "center", justifyContent: "center", padding: 20, fontFamily: "system-ui,sans-serif" }}>
      <div style={{ background: "#0f172a", border: "1px solid #1e293b", borderRadius: 18, padding: "30px 24px", maxWidth: 430, width: "100%" }}>
        <div style={{ textAlign: "center", marginBottom: 22 }}>
          <div style={{ fontSize: 44, marginBottom: 6 }}>⚽</div>
          <div style={{ fontSize: 28, fontWeight: 900, letterSpacing: 5, color: "#22c55e" }}>LIGA MARTES</div>
          <div style={{ fontSize: 9, color: "#334155", letterSpacing: 3, marginTop: 5 }}>PLATAFORMA MULTI-TORNEO</div>
          <div style={{ width: 32, height: 2, background: "#22c55e", margin: "10px auto 0", borderRadius: 2 }} />
        </div>
        {!mode ? (
          <div>
            <div style={{ fontSize: 9, letterSpacing: 3, color: "#334155", textAlign: "center", marginBottom: 11 }}>¿CÓMO QUIERES INGRESAR?</div>
            {[{ m: "admin", icon: "🛡️", title: "ADMINISTRADOR", sub: "Gestionar torneos · Jugadores · Resultados", col: "#22c55e" },
              { m: "player", icon: "👀", title: "VER RANKING", sub: "Consultar estadísticas · KPIs · Historial", col: "#3b82f6" }
            ].map(o => (
              <button key={o.m} onClick={() => o.m === "player" ? onLogin("player") : setMode("admin")}
                style={{ width: "100%", background: "#0f172a", border: "2px solid #1e293b", borderRadius: 12, padding: 13, cursor: "pointer", display: "flex", alignItems: "center", gap: 12, marginBottom: 9, textAlign: "left", color: "#e2e8f0" }}>
                <div style={{ width: 38, height: 38, borderRadius: 9, background: o.col + "18", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 20 }}>{o.icon}</div>
                <div style={{ flex: 1 }}>
                  <div style={{ fontSize: 13, fontWeight: 900, letterSpacing: 2, color: o.col, marginBottom: 2 }}>{o.title}</div>
                  <div style={{ fontSize: 11, color: "#475569" }}>{o.sub}</div>
                </div>
                <span style={{ fontSize: 20, color: o.col }}>›</span>
              </button>
            ))}
          </div>
        ) : (
          <div>
            <button onClick={() => { setMode(null); setErr(""); }} style={{ background: "transparent", border: "none", color: "#64748b", cursor: "pointer", fontSize: 13, padding: "0 0 14px", display: "block" }}>← Volver</button>
            <div style={{ display: "flex", alignItems: "center", gap: 10, marginBottom: 15 }}>
              <span style={{ fontSize: 24 }}>🛡️</span>
              <div>
                <div style={{ fontSize: 17, fontWeight: 900, letterSpacing: 2, color: "#e2e8f0" }}>ACCESO ADMIN</div>
                <div style={{ fontSize: 12, color: "#475569" }}>Credenciales de administrador</div>
              </div>
            </div>
            <input style={{ ...ST.field, marginBottom: 8 }} type="text" placeholder="Usuario" value={user} onChange={e => { setUser(e.target.value); setErr(""); }} />
            <input style={{ ...ST.field, marginBottom: 8, letterSpacing: 4 }} type="password" placeholder="Contraseña" value={pwd} onChange={e => { setPwd(e.target.value); setErr(""); }} onKeyDown={e => e.key === "Enter" && doLogin()} />
            {err && <div style={{ color: "#ef4444", fontSize: 13, marginBottom: 8 }}>⚠ {err}</div>}
            <button style={{ ...ST.btnGreen, width: "100%", padding: 12, fontSize: 14, letterSpacing: 3, marginBottom: 10 }} onClick={doLogin}>INGRESAR →</button>
            <div style={{ fontSize: 11, color: "#334155", textAlign: "center" }}>Usuario: <code style={{ color: "#22c55e" }}>admin</code> · Clave: <code style={{ color: "#22c55e" }}>gol2025</code></div>
          </div>
        )}
      </div>
    </div>
  );
}

export default function App() {
  const [role, setRole] = useState(null);
  const [tab, setTab] = useState("ranking");
  const [torneos, setTorneos] = useState([]);
  const [jugadores, setJugadores] = useState([]);
  const [participaciones, setParticipaciones] = useState([]);
  const [partidos, setPartidos] = useState([]);
  const [activoId, setActivoId] = useState(null);
  const [vistaId, setVistaId] = useState(null);
  const [loading, setLoading] = useState(true);
  const [sel, setSel] = useState([]);
  const [teams, setTeams] = useState(null);
  const [matchResult, setMatchResult] = useState(null);
  const [manualMode, setManualMode] = useState(false);
  const [mA, setMA] = useState([]);
  const [mB, setMB] = useState([]);
  const [newPlayerName, setNewPlayerName] = useState("");
  const [editPlayer, setEditPlayer] = useState(null);
  const [newTName, setNewTName] = useState("");
  const [newTWeeks, setNewTWeeks] = useState(12);
  const [showNewT, setShowNewT] = useState(false);
  const [modal, setModal] = useState(null);
  const [dashFilter, setDashFilter] = useState("current");

  useEffect(() => { if (role) loadAll(); }, [role]);

  async function loadAll() {
    setLoading(true);
    const [{ data: t }, { data: j }, { data: p }, { data: pa }] = await Promise.all([
      supabase.from("torneos").select("*").order("id"),
      supabase.from("jugadores").select("*").order("nombre"),
      supabase.from("participaciones").select("*"),
      supabase.from("partidos").select("*").order("semana"),
    ]);
    setTorneos(t || []);
    setJugadores(j || []);
    setParticipaciones(p || []);
    setPartidos(pa || []);
    const activo = (t || []).find(x => x.estado === "active");
    setActivoId(activo?.id || null);
    setVistaId(activo?.id || (t || [])[0]?.id || null);
    setLoading(false);
  }

  const torneo = torneos.find(t => t.id === vistaId);
  const torneoActivo = torneos.find(t => t.id === activoId);

  const statsVista = useMemo(() => {
    if (!torneo) return [];
    const parts = participaciones.filter(p => p.torneo_id === torneo.id);
    const pars = partidos.filter(p => p.torneo_id === torneo.id);
    return parts.map(p => {
      const jug = jugadores.find(j => j.id === p.jugador_id);
      return { ...calcStats(p, pars), nombre: jug?.nombre || "?" };
    });
  }, [torneo, participaciones, partidos, jugadores]);

  const statsActivo = useMemo(() => {
    if (!torneoActivo) return [];
    const parts = participaciones.filter(p => p.torneo_id === torneoActivo.id);
    const pars = partidos.filter(p => p.torneo_id === torneoActivo.id);
    return parts.map(p => {
      const jug = jugadores.find(j => j.id === p.jugador_id);
      return { ...calcStats(p, pars), nombre: jug?.nombre || "?" };
    });
  }, [torneoActivo, participaciones, partidos, jugadores]);

  const rankVista = useMemo(() => ranked(statsVista), [statsVista]);
  const rankActivo = useMemo(() => ranked(statsActivo), [statsActivo]);

  function nameOf(id) { return jugadores.find(j => j.id === id)?.nombre || "?"; }

  async function addPlayer() {
    const nombre = newPlayerName.trim();
    if (!nombre || !torneoActivo) return;
    const { data: j } = await supabase.from("jugadores").insert({ nombre }).select().single();
    if (j) {
      await supabase.from("participaciones").insert({ torneo_id: torneoActivo.id, jugador_id: j.id });
      setNewPlayerName("");
      loadAll();
    }
  }

  async function saveEditPlayer() {
    if (!editPlayer) return;
    await supabase.from("jugadores").update({ nombre: editPlayer.nombre }).eq("id", editPlayer.id);
    setEditPlayer(null);
    loadAll();
  }

  async function deletePlayer(jId) {
    if (!confirm("¿Eliminar jugador?")) return;
    await supabase.from("participaciones").delete().eq("jugador_id", jId).eq("torneo_id", activoId);
    loadAll();
  }

  async function confirmMatch() {
    if (!matchResult || !teams || !torneoActivo) return;
    const match = {
      torneo_id: torneoActivo.id,
      semana: torneoActivo.semana_actual,
      fecha: new Date().toISOString().split("T")[0],
      equipo_a: teams.A.map(p => p.jugador_id),
      equipo_b: teams.B.map(p => p.jugador_id),
      resultado: matchResult,
      contexto: ""
    };
    await supabase.from("partidos").insert(match);
    const allIds = [...match.equipo_a, ...match.equipo_b];
    for (const jId of allIds) {
      const part = participaciones.find(p => p.torneo_id === activoId && p.jugador_id === jId);
      if (!part) continue;
      const inA = match.equipo_a.includes(jId);
      let pg = part.pg, pe = part.pe, pp = part.pp;
      if (matchResult === "draw") pe++;
      else if ((matchResult === "A" && inA) || (matchResult === "B" && !inA)) pg++;
      else pp++;
      await supabase.from("participaciones").update({ pg, pe, pp }).eq("id", part.id);
    }
    await supabase.from("torneos").update({ semana_actual: torneoActivo.semana_actual + 1 }).eq("id", activoId);
    setTeams(null); setSel([]); setMatchResult(null); setMA([]); setMB([]); setTab("ranking");
    loadAll();
  }

  async function fixMatch(matchId, newRes) {
    const match = partidos.find(m => m.id === matchId);
    if (!match) return;
    const oldRes = match.resultado;
    await supabase.from("partidos").update({ resultado: newRes }).eq("id", matchId);
    const allIds = [...match.equipo_a, ...match.equipo_b];
    for (const jId of allIds) {
      const part = participaciones.find(p => p.torneo_id === match.torneo_id && p.jugador_id === jId);
      if (!part) continue;
      const inA = match.equipo_a.includes(jId);
      let pg = part.pg, pe = part.pe, pp = part.pp;
      if (oldRes === "draw") pe--; else if ((oldRes === "A" && inA) || (oldRes === "B" && !inA)) pg--; else pp--;
      if (newRes === "draw") pe++; else if ((newRes === "A" && inA) || (newRes === "B" && !inA)) pg++; else pp++;
      await supabase.from("participaciones").update({ pg, pe, pp }).eq("id", part.id);
    }
    setModal(null); loadAll();
  }

  async function deleteMatch(matchId) {
    if (!confirm("¿Eliminar este partido?")) return;
    const match = partidos.find(m => m.id === matchId);
    if (!match) return;
    const allIds = [...match.equipo_a, ...match.equipo_b];
    for (const jId of allIds) {
      const part = participaciones.find(p => p.torneo_id === match.torneo_id && p.jugador_id === jId);
      if (!part) continue;
      const inA = match.equipo_a.includes(jId);
      let pg = part.pg, pe = part.pe, pp = part.pp;
      if (match.resultado === "draw") pe--; else if ((match.resultado === "A" && inA) || (match.resultado === "B" && !inA)) pg--; else pp--;
      await supabase.from("participaciones").update({ pg: Math.max(0,pg), pe: Math.max(0,pe), pp: Math.max(0,pp) }).eq("id", part.id);
    }
    await supabase.from("partidos").delete().eq("id", matchId);
    await supabase.from("torneos").update({ semana_actual: Math.max(1, torneoActivo.semana_actual - 1) }).eq("id", activoId);
    loadAll();
  }

  async function createTournament() {
    const nombre = newTName.trim();
    if (!nombre) return;
    const { data: t } = await supabase.from("torneos").insert({ nombre, estado: "active", semana_actual: 1, total_semanas: newTWeeks }).select().single();
    if (t) {
      const jugsActuales = participaciones.filter(p => p.torneo_id === activoId).map(p => p.jugador_id);
      for (const jId of jugsActuales) {
        await supabase.from("participaciones").insert({ torneo_id: t.id, jugador_id: jId });
      }
      setActivoId(t.id); setVistaId(t.id); setShowNewT(false); setNewTName(""); setTab("ranking");
      loadAll();
    }
  }

  async function closeTournament(tId) {
    await supabase.from("torneos").update({ estado: "closed" }).eq("id", tId);
    if (activoId === tId) setActivoId(null);
    setModal(null); loadAll();
  }

  const isAdmin = role === "admin";
  const TABS = isAdmin
    ? [{ k: "ranking", l: "🏆 Ranking" }, { k: "dashboard", l: "📊 Dashboard" }, { k: "partido", l: "⚽ Partido" }, { k: "historial", l: "📅 Historial" }, { k: "torneos", l: "🏅 Torneos" }, { k: "jugadores", l: "👤 Jugadores" }]
    : [{ k: "ranking", l: "🏆 Ranking" }, { k: "dashboard", l: "📊 Dashboard" }, { k: "historial", l: "📅 Historial" }, { k: "torneos", l: "🏅 Torneos" }];

  if (!role) return <LoginScreen onLogin={r => { setRole(r); setTab("ranking"); }} />;
  if (loading) return <div style={{ ...ST.root, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 18, color: "#22c55e" }}>⚽ Cargando Liga Martes...</div>;

  return (
    <div style={ST.root}>
      <div style={ST.hdr}>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <span style={{ fontSize: 22 }}>⚽</span>
          <div>
            <div style={{ fontSize: 17, fontWeight: 900, letterSpacing: 3, color: "#22c55e", lineHeight: 1 }}>LIGA MARTES</div>
            <div style={{ fontSize: 10, color: "#64748b", letterSpacing: 1, marginTop: 2 }}>
              <span style={{ color: isAdmin ? "#fbbf24" : "#60a5fa" }}>{isAdmin ? "🛡️ Admin" : "👀 Espectador"}</span>
              {torneo ? ` · ${torneo.nombre} · S${torneo.semana_actual}/${torneo.total_semanas}` : ""}
            </div>
          </div>
        </div>
        <button style={ST.btnGhost} onClick={() => setRole(null)}>⏻ Salir</button>
      </div>

      <div style={ST.nav}>
        {TABS.map(t => (
          <button key={t.k} onClick={() => setTab(t.k)}
            style={{ background: "transparent", border: "none", borderBottom: tab === t.k ? "2px solid #22c55e" : "2px solid transparent", color: tab === t.k ? "#22c55e" : "#64748b", padding: "9px 13px", cursor: "pointer", fontSize: 13, fontWeight: 700, whiteSpace: "nowrap" }}>
            {t.l}
          </button>
        ))}
      </div>

      <div style={ST.main}>

        {tab === "ranking" && (
          <div>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 12, flexWrap: "wrap", gap: 8 }}>
              <div>
                <div style={{ fontSize: 15, fontWeight: 900, letterSpacing: 2 }}>TABLA DE POSICIONES</div>
                <div style={{ fontSize: 11, color: "#64748b", marginTop: 2 }}>{torneo?.nombre} · {rankVista.length} jugadores</div>
              </div>
              <span style={{ ...ST.tag, background: torneo?.estado === "active" ? "#22c55e22" : "#f59e0b22", color: torneo?.estado === "active" ? "#22c55e" : "#f59e0b" }}>
                {torneo?.estado === "active" ? "🟢 En curso" : "🏁 Finalizado"}
              </span>
            </div>
            {!rankVista.length ? <Empty text="Sin jugadores en este torneo" /> : (
              <div style={{ overflowX: "auto" }}>
                <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 13 }}>
                  <thead><tr>
                    {["#", "Jugador", "PTS", "PJ", "EF%", "G", "E", "P", "Racha", "Forma"].map((h, i) => (
                      <th key={i} style={{ padding: "7px 8px", fontSize: 10, color: "#475569", letterSpacing: 2, fontWeight: 700, borderBottom: "1px solid #1e293b", textAlign: i <= 1 ? "left" : "center" }}>{h}</th>
                    ))}
                  </tr></thead>
                  <tbody>
                    {rankVista.map((p, i) => (
                      <tr key={p.id} style={{ borderBottom: "1px solid #0d1523", background: i === 0 && p.pj > 0 ? "rgba(34,197,94,.05)" : "transparent" }}>
                        <td style={{ padding: "8px", color: "#475569", fontWeight: 700, fontSize: 12 }}>{i === 0 && p.pj > 0 ? "👑" : i === 1 && p.pj > 0 ? "🥈" : i === 2 && p.pj > 0 ? "🥉" : i + 1}</td>
                        <td style={{ padding: "8px", fontWeight: 700, color: "#e2e8f0" }}>{p.nombre}</td>
                        <td style={{ padding: "8px", textAlign: "center", fontWeight: 900, fontSize: 16, color: "#22c55e" }}>{p.pts}</td>
                        <td style={{ padding: "8px", textAlign: "center", color: "#94a3b8" }}>{p.pj}</td>
                        <td style={{ padding: "8px", textAlign: "center" }}><span style={{ ...ST.tag, background: effColor(p.eff) + "22", color: effColor(p.eff) }}>{p.eff}%</span></td>
                        <td style={{ padding: "8px", textAlign: "center", color: "#4ade80" }}>{p.pg}</td>
                        <td style={{ padding: "8px", textAlign: "center", color: "#fbbf24" }}>{p.pe}</td>
                        <td style={{ padding: "8px", textAlign: "center", color: "#f87171" }}>{p.pp}</td>
                        <td style={{ padding: "8px", textAlign: "center", fontSize: 11, color: p.curWin > 1 ? "#22c55e" : p.curLoss > 1 ? "#ef4444" : "#475569" }}>
                          {p.pj > 0 ? (p.curWin > 0 ? `🔥${p.curWin}V` : p.curLoss > 0 ? `❄️${p.curLoss}D` : "—") : "—"}
                        </td>
                        <td style={{ padding: "8px", textAlign: "center" }}>
                          <div style={{ display: "flex", gap: 2, justifyContent: "center" }}>{p.recent.map((r, ri) => <Bubble key={ri} r={r} />)}</div>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            )}
          </div>
        )}

        {tab === "dashboard" && (
          <div>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 14, flexWrap: "wrap", gap: 8 }}>
              <div style={{ fontSize: 15, fontWeight: 900, letterSpacing: 2 }}>ESTADÍSTICAS & KPIs</div>
              <div style={{ display: "flex", gap: 6 }}>
                {["current", "all"].map(f => (
                  <button key={f} onClick={() => setDashFilter(f)}
                    style={{ ...ST.btnGhost, color: dashFilter === f ? "#22c55e" : "#94a3b8", borderColor: dashFilter === f ? "#22c55e" : "#334155", fontSize: 11, padding: "4px 10px" }}>
                    {f === "current" ? "🏆 Torneo actual" : "🌍 All-time"}
                  </button>
                ))}
              </div>
            </div>
            {dashFilter === "current" ? (
              <>
                {!rankVista.filter(p => p.pj > 0).length ? <Empty text="Juega el primer partido 📊" /> : (() => {
                  const wm = rankVista.filter(p => p.pj > 0);
                  return (
                    <>
                      <div style={ST.kpiGrid}>
                        <KpiCard icon="🏅" label="MEJOR EFF." val={[...wm].sort((a,b)=>b.eff-a.eff)[0].nombre} sub={`${[...wm].sort((a,b)=>b.eff-a.eff)[0].eff}%`} accent="#22c55e" />
                        <KpiCard icon="👑" label="LÍDER" val={wm[0].nombre} sub={`${wm[0].pts} puntos`} accent="#fbbf24" />
                        <KpiCard icon="🏆" label="MÁS VICTORIAS" val={[...wm].sort((a,b)=>b.pg-a.pg)[0].nombre} sub={`${[...wm].sort((a,b)=>b.pg-a.pg)[0].pg} victorias`} accent="#f59e0b" />
                        <KpiCard icon="⚽" label="MÁS ACTIVO" val={[...wm].sort((a,b)=>b.pj-a.pj)[0].nombre} sub={`${[...wm].sort((a,b)=>b.pj-a.pj)[0].pj} partidos`} accent="#3b82f6" />
                        <KpiCard icon="🔥" label="MEJOR RACHA" val={[...wm].sort((a,b)=>b.maxWin-a.maxWin)[0].nombre} sub={`${[...wm].sort((a,b)=>b.maxWin-a.maxWin)[0].maxWin} seguidas`} accent="#f97316" />
                        <KpiCard icon="❄️" label="PEOR RACHA" val={[...wm].sort((a,b)=>b.maxLoss-a.maxLoss)[0].nombre} sub={`${[...wm].sort((a,b)=>b.maxLoss-a.maxLoss)[0].maxLoss} derrotas`} accent="#8b5cf6" />
                        <KpiCard icon="😓" label="MÁS DERROTAS" val={[...wm].sort((a,b)=>b.pp-a.pp)[0].nombre} sub={`${[...wm].sort((a,b)=>b.pp-a.pp)[0].pp} derrotas`} accent="#ef4444" />
                        <KpiCard icon="📅" label="PARTIDOS" val={partidos.filter(p=>p.torneo_id===vistaId).length} sub="jugados" accent="#06b6d4" />
                      </div>
                      <div style={ST.card}>
                        <div style={{ fontSize: 10, color: "#64748b", letterSpacing: 2, marginBottom: 12 }}>EFICIENCIA POR JUGADOR</div>
                        {wm.map(p => (
                          <div key={p.id} style={{ marginBottom: 9 }}>
                            <div style={{ display: "flex", justifyContent: "space-between", fontSize: 12, marginBottom: 2 }}>
                              <span style={{ fontWeight: 700, color: "#e2e8f0" }}>{p.nombre}</span>
                              <span style={{ color: "#64748b" }}>{p.pts}pts · {p.eff}% · {p.pg}V {p.pe}E {p.pp}D</span>
                            </div>
                            <div style={{ background: "#0f172a", borderRadius: 6, height: 10, overflow: "hidden" }}>
                              <div style={{ width: `${p.eff}%`, height: 10, background: effColor(p.eff), borderRadius: 6 }} />
                            </div>
                          </div>
                        ))}
                      </div>
                    </>
                  );
                })()}
              </>
            ) : (
              <div style={ST.card}>
                <div style={{ fontSize: 10, color: "#64748b", letterSpacing: 2, marginBottom: 12 }}>RANKING ALL-TIME · {torneos.length} torneos</div>
                <div style={{ overflowX: "auto" }}>
                  <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 13 }}>
                    <thead><tr>
                      {["#", "Jugador", "PTS", "PJ", "PG", "EFF%", "Torneos"].map((h, i) => (
                        <th key={i} style={{ padding: "6px 8px", fontSize: 10, color: "#475569", textAlign: i <= 1 ? "left" : "center", letterSpacing: 1 }}>{h}</th>
                      ))}
                    </tr></thead>
                    <tbody>
                      {(() => {
                        const map = {};
                        torneos.forEach(t => {
                          const parts = participaciones.filter(p => p.torneo_id === t.id);
                          parts.forEach(p => {
                            const nombre = jugadores.find(j => j.id === p.jugador_id)?.nombre || "?";
                            if (!map[nombre]) map[nombre] = { nombre, pts: 0, pj: 0, pg: 0, pe: 0, pp: 0, count: 0 };
                            const pts = p.pg * 3 + p.pe;
                            const pj = p.pg + p.pe + p.pp;
                            map[nombre].pts += pts; map[nombre].pj += pj;
                            map[nombre].pg += p.pg; map[nombre].pe += p.pe; map[nombre].pp += p.pp;
                            if (pj > 0) map[nombre].count++;
                          });
                        });
                        return Object.values(map).sort((a, b) => b.pts - a.pts).map((p, i) => {
                          const eff = p.pj ? Math.round((p.pts / (p.pj * 3)) * 100) : 0;
                          return (
                            <tr key={i} style={{ borderBottom: "1px solid #0d1523" }}>
                              <td style={{ padding: "8px", color: "#475569", fontSize: 12 }}>{i + 1}</td>
                              <td style={{ padding: "8px", fontWeight: 700 }}>{p.nombre}</td>
                              <td style={{ padding: "8px", textAlign: "center", color: "#22c55e", fontWeight: 900 }}>{p.pts}</td>
                              <td style={{ padding: "8px", textAlign: "center", color: "#94a3b8" }}>{p.pj}</td>
                              <td style={{ padding: "8px", textAlign: "center", color: "#4ade80" }}>{p.pg}</td>
                              <td style={{ padding: "8px", textAlign: "center" }}><span style={{ ...ST.tag, background: effColor(eff) + "22", color: effColor(eff) }}>{eff}%</span></td>
                              <td style={{ padding: "8px", textAlign: "center", color: "#94a3b8" }}>{p.count}</td>
                            </tr>
                          );
                        });
                      })()}
                    </tbody>
                  </table>
                </div>
              </div>
            )}
          </div>
        )}

        {tab === "partido" && isAdmin && (
          <div>
            <div style={{ fontSize: 15, fontWeight: 900, letterSpacing: 2, marginBottom: 12 }}>ARMAR PARTIDO · SEMANA {torneoActivo?.semana_actual}</div>
            {!torneoActivo ? <Empty text="No hay torneo activo. Crea uno en 🏅 Torneos" /> : !teams ? (
              <>
                <div style={{ ...ST.card, marginBottom: 12 }}>
                  <strong style={{ color: "#e2e8f0" }}>Selecciona 12 jugadores</strong> que confirmaron esta semana.
                  <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginTop: 8 }}>
                    <span style={{ fontWeight: 900, fontSize: 22, color: sel.length === 12 ? "#22c55e" : "#e2e8f0" }}>{sel.length}/12</span>
                    <div style={{ display: "flex", gap: 6 }}>
                      <button style={{ ...ST.btnGhost, fontSize: 11, padding: "4px 9px" }} onClick={() => { setSel([]); setMA([]); setMB([]); }}>Limpiar</button>
                      <button style={{ ...ST.btnGhost, fontSize: 11, padding: "4px 9px", color: manualMode ? "#3b82f6" : "#94a3b8", borderColor: manualMode ? "#3b82f6" : "#334155" }} onClick={() => setManualMode(m => !m)}>{manualMode ? "✓ Manual" : "Manual"}</button>
                    </div>
                  </div>
                </div>
                <div style={ST.pgrid}>
                  {[...statsActivo].sort((a, b) => a.nombre.localeCompare(b.nombre)).map(p => {
                    const s = sel.includes(p.jugador_id);
                    return (
                      <button key={p.jugador_id} onClick={() => setSel(prev => prev.includes(p.jugador_id) ? prev.filter(x => x !== p.jugador_id) : prev.length < 12 ? [...prev, p.jugador_id] : prev)}
                        style={{ background: s ? "#14532d" : "#1e293b", border: `2px solid ${s ? "#22c55e" : "#334155"}`, borderRadius: 9, padding: "9px 8px", cursor: "pointer", textAlign: "center" }}>
                        <div style={{ fontSize: 12, fontWeight: 700, color: "#e2e8f0", marginBottom: 2 }}>{p.nombre}</div>
                        <div style={{ fontSize: 10, color: "#64748b" }}>{p.pts}pts · {p.eff}%</div>
                      </button>
                    );
                  })}
                </div>
                {!manualMode && sel.length === 12 && (
                  <button style={{ ...ST.btnGreen, width: "100%", padding: 12, fontSize: 14, letterSpacing: 2 }}
                    onClick={() => { const pool = statsActivo.filter(p => sel.includes(p.jugador_id)); setTeams(snakeDraft(pool)); setMatchResult(null); }}>
                    ⚽ Armar Equipos Automáticamente
                  </button>
                )}
                {manualMode && sel.length === 12 && (
                  <div>
                    <div style={{ display: "flex", gap: 8, marginBottom: 10 }}>
                      {[["A", "#22c55e", mA, setMA], ["B", "#3b82f6", mB, setMB]].map(([tm, col, list, setList]) => (
                        <div key={tm} style={{ flex: 1, background: "#1e293b", borderRadius: 9, overflow: "hidden" }}>
                          <div style={{ background: col, padding: "7px 11px", fontWeight: 900, fontSize: 12, letterSpacing: 2, color: "#050d18" }}>EQUIPO {tm} ({list.length}/6)</div>
                          {sel.map(jId => {
                            const p = statsActivo.find(x => x.jugador_id === jId);
                            const here = list.includes(jId);
                            const taken = (tm === "A" ? mB : mA).includes(jId);
                            return (
                              <button key={jId} disabled={taken}
                                onClick={() => setList(prev => prev.includes(jId) ? prev.filter(x => x !== jId) : prev.length < 6 ? [...prev, jId] : prev)}
                                style={{ width: "100%", background: here ? col + "15" : "transparent", border: "none", borderBottom: "1px solid #0f172a", padding: "7px 11px", textAlign: "left", cursor: taken ? "not-allowed" : "pointer", color: taken ? "#334155" : here ? col : "#94a3b8", fontSize: 12, fontWeight: here ? 700 : 400 }}>
                                {here ? "✓ " : ""}{p?.nombre}
                              </button>
                            );
                          })}
                        </div>
                      ))}
                    </div>
                    {mA.length === 6 && mB.length === 6 && (
                      <button style={{ ...ST.btnGreen, width: "100%", padding: 12, fontSize: 14 }}
                        onClick={() => { const pool = statsActivo.filter(p => sel.includes(p.jugador_id)); setTeams({ A: mA.map(id => pool.find(p => p.jugador_id === id)), B: mB.map(id => pool.find(p => p.jugador_id === id)) }); setManualMode(false); setMatchResult(null); }}>
                        ✓ Confirmar Equipos
                      </button>
                    )}
                  </div>
                )}
              </>
            ) : (
              <>
                <div style={{ display: "flex", gap: 8, marginBottom: 12 }}>
                  {[["A", "#22c55e", teams.A], ["B", "#3b82f6", teams.B]].map(([tm, col, list]) => (
                    <div key={tm} style={{ flex: 1, background: "#1e293b", borderRadius: 9, overflow: "hidden" }}>
                      <div style={{ background: col, padding: "8px 11px", fontWeight: 900, fontSize: 12, letterSpacing: 2, color: "#050d18" }}>EQUIPO {tm}</div>
                      {list.map(p => <div key={p.jugador_id} style={{ display: "flex", justifyContent: "space-between", padding: "7px 11px", borderBottom: "1px solid #0f172a", fontSize: 13 }}><span>{p.nombre}</span><span style={{ color: "#475569", fontSize: 11 }}>{p.pts}p</span></div>)}
                    </div>
                  ))}
                </div>
                <div style={ST.card}>
                  <div style={{ fontSize: 10, letterSpacing: 2, color: "#64748b", marginBottom: 10, textAlign: "center" }}>¿QUIÉN GANÓ?</div>
                  <div style={{ display: "flex", gap: 7, marginBottom: 12, flexWrap: "wrap" }}>
                    {[["A", "🟢 Ganó Equipo A", "#16a34a"], ["draw", "🟡 Empate", "#ca8a04"], ["B", "🔵 Ganó Equipo B", "#1d4ed8"]].map(([v, label, col]) => (
                      <button key={v} onClick={() => setMatchResult(v)}
                        style={{ flex: 1, background: matchResult === v ? col : "transparent", border: `2px solid ${matchResult === v ? col : "#334155"}`, color: matchResult === v ? "#fff" : "#e2e8f0", borderRadius: 9, padding: "10px 7px", cursor: "pointer", fontWeight: 700, fontSize: 12, minWidth: 85 }}>
                        {label}
                      </button>
                    ))}
                  </div>
                  <div style={{ display: "flex", gap: 7 }}>
                    <button style={ST.btnGhost} onClick={() => { setTeams(null); setSel([]); setMatchResult(null); setMA([]); setMB([]); }}>← Volver</button>
                    {matchResult && <button style={{ ...ST.btnGreen, flex: 1, padding: 11, fontSize: 13, letterSpacing: 1 }} onClick={confirmMatch}>✅ Confirmar Resultado</button>}
                  </div>
                </div>
              </>
            )}
          </div>
        )}

        {tab === "historial" && (
          <div>
            <div style={{ fontSize: 15, fontWeight: 900, letterSpacing: 2, marginBottom: 12 }}>HISTORIAL DE PARTIDOS</div>
            {!partidos.filter(p => p.torneo_id === vistaId).length ? <Empty text="Sin partidos registrados" /> :
              [...partidos.filter(p => p.torneo_id === vistaId)].reverse().map(m => (
                <div key={m.id} style={{ ...ST.card, padding: 0, overflow: "hidden" }}>
                  <div style={{ display: "flex", alignItems: "center", gap: 8, padding: "9px 13px", borderBottom: "1px solid #0d1523", flexWrap: "wrap" }}>
                    <span style={{ fontWeight: 900, color: "#22c55e", fontSize: 13 }}>Semana {m.semana}</span>
                    <span style={{ color: "#475569", fontSize: 11, flex: 1 }}>{m.fecha}</span>
                    {m.contexto && <span style={{ fontSize: 11, color: "#64748b", fontStyle: "italic" }}>{m.contexto}</span>}
                    <span style={{ fontSize: 13, fontWeight: 600 }}>{m.resultado === "draw" ? "🟡 Empate" : m.resultado === "A" ? "🟢 Ganó A" : "🔵 Ganó B"}</span>
                    {isAdmin && torneo?.estado === "active" && (
                      <div style={{ display: "flex", gap: 5 }}>
                        <button style={ST.btnBlue} onClick={() => setModal({ type: "edit-match", id: m.id })}>✏️</button>
                        <button style={ST.btnDanger} onClick={() => deleteMatch(m.id)}>🗑️</button>
                      </div>
                    )}
                  </div>
                  <div style={{ display: "flex", padding: "10px 13px", gap: 16 }}>
                    {[["A", "#4ade80", m.equipo_a], ["B", "#60a5fa", m.equipo_b]].map(([tm, col, ids]) => (
                      <div key={tm} style={{ flex: 1 }}>
                        <div style={{ fontWeight: 800, fontSize: 10, letterSpacing: 2, color: col, marginBottom: 5 }}>EQUIPO {tm}</div>
                        {ids.map(id => <div key={id} style={{ fontSize: 12, color: "#94a3b8", padding: "1px 0" }}>{nameOf(id)}</div>)}
                      </div>
                    ))}
                  </div>
                </div>
              ))
            }
          </div>
        )}

        {tab === "torneos" && (
          <div>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 14 }}>
              <div style={{ fontSize: 15, fontWeight: 900, letterSpacing: 2 }}>GESTIÓN DE TORNEOS</div>
              {isAdmin && <button style={ST.btnGreen} onClick={() => setShowNewT(true)}>+ Nuevo Torneo</button>}
            </div>
            {isAdmin && showNewT && (
              <div style={{ ...ST.card, border: "1px solid #22c55e44", marginBottom: 14 }}>
                <div style={{ fontSize: 13, fontWeight: 700, color: "#22c55e", marginBottom: 12 }}>➕ Crear Nuevo Torneo</div>
                <div style={{ display: "flex", gap: 8, marginBottom: 10, flexWrap: "wrap" }}>
                  <input style={{ ...ST.field, flex: 2, minWidth: 180 }} placeholder="Ej: Clausura 2026" value={newTName} onChange={e => setNewTName(e.target.value)} />
                  <input style={{ ...ST.field, flex: 1, minWidth: 80, maxWidth: 120 }} type="number" placeholder="Fechas" value={newTWeeks} onChange={e => setNewTWeeks(Number(e.target.value))} />
                </div>
                <div style={{ fontSize: 11, color: "#475569", marginBottom: 10 }}>Los jugadores del torneo activo se migran automáticamente.</div>
                <div style={{ display: "flex", gap: 7 }}>
                  <button style={ST.btnGhost} onClick={() => setShowNewT(false)}>Cancelar</button>
                  <button style={ST.btnGreen} onClick={createTournament}>Crear Torneo</button>
                </div>
              </div>
            )}
            {torneos.map(t => {
              const tStats = participaciones.filter(p => p.torneo_id === t.id).map(p => ({ ...p, pts: p.pg * 3 + p.pe, pj: p.pg + p.pe + p.pp }));
              const lider = [...tStats].sort((a, b) => b.pts - a.pts)[0];
              const liderNombre = lider ? jugadores.find(j => j.id === lider.jugador_id)?.nombre : null;
              const isVista = vistaId === t.id;
              return (
                <div key={t.id} onClick={() => setVistaId(t.id)}
                  style={{ ...ST.card, cursor: "pointer", border: isVista ? "2px solid #22c55e" : "2px solid transparent" }}>
                  <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", flexWrap: "wrap", gap: 8 }}>
                    <div>
                      <div style={{ fontSize: 15, fontWeight: 900, color: "#e2e8f0" }}>{t.nombre}</div>
                      <div style={{ fontSize: 11, color: "#64748b", marginTop: 2 }}>{partidos.filter(p => p.torneo_id === t.id).length} partidos · {tStats.length} jugadores · S{t.semana_actual - 1}/{t.total_semanas}</div>
                      {liderNombre && <div style={{ fontSize: 11, color: "#fbbf24", marginTop: 3 }}>👑 Líder: {liderNombre} ({lider.pts}pts)</div>}
                    </div>
                    <div style={{ display: "flex", gap: 6, alignItems: "center", flexWrap: "wrap" }}>
                      <span style={{ ...ST.tag, background: t.estado === "active" ? "#22c55e22" : "#f59e0b22", color: t.estado === "active" ? "#22c55e" : "#f59e0b" }}>{t.estado === "active" ? "🟢 En curso" : "🏁 Finalizado"}</span>
                      {isVista && <span style={{ ...ST.tag, background: "#3b82f622", color: "#3b82f6" }}>👁 Viendo</span>}
                      {isAdmin && t.estado === "active" && <button style={ST.btnDanger} onClick={e => { e.stopPropagation(); setModal({ type: "close-t", id: t.id }); }}>🏁 Cerrar</button>}
                    </div>
                  </div>
                </div>
              );
            })}
          </div>
        )}

        {tab === "jugadores" && isAdmin && (
          <div>
            <div style={{ fontSize: 15, fontWeight: 900, letterSpacing: 2, marginBottom: 12 }}>GESTIÓN DE JUGADORES</div>
            <div style={{ display: "flex", gap: 8, marginBottom: 12 }}>
              <input style={{ ...ST.field, flex: 1 }} placeholder="Nombre del jugador..." value={newPlayerName} onChange={e => setNewPlayerName(e.target.value)} onKeyDown={e => e.key === "Enter" && addPlayer()} />
              <button style={ST.btnGreen} onClick={addPlayer}>+ Agregar</button>
            </div>
            {editPlayer && (
              <div style={{ background: "#1e3a5f", border: "1px solid #3b82f6", borderRadius: 9, padding: 11, marginBottom: 11, display: "flex", gap: 7, alignItems: "center" }}>
                <input style={{ ...ST.field, flex: 1 }} value={editPlayer.nombre} autoFocus onChange={e => setEditPlayer(ep => ({ ...ep, nombre: e.target.value }))} onKeyDown={e => { if (e.key === "Enter") saveEditPlayer(); if (e.key === "Escape") setEditPlayer(null); }} />
                <button style={{ ...ST.btnGreen, background: "#3b82f6" }} onClick={saveEditPlayer}>Guardar</button>
                <button style={ST.btnGhost} onClick={() => setEditPlayer(null)}>✕</button>
              </div>
            )}
            <div style={{ fontSize: 10, color: "#475569", letterSpacing: 2, marginBottom: 8 }}>{statsActivo.length} JUGADORES</div>
            {[...statsActivo].sort((a, b) => b.pts - a.pts).map(p => (
              <div key={p.jugador_id} style={{ ...ST.card, display: "flex", alignItems: "center", gap: 11, marginBottom: 7 }}>
                <div style={{ width: 32, height: 32, borderRadius: "50%", background: "#22c55e18", display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 900, fontSize: 12, color: "#22c55e", flexShrink: 0 }}>{p.nombre.charAt(0).toUpperCase()}</div>
                <div style={{ flex: 1 }}>
                  <div style={{ fontWeight: 700, color: "#e2e8f0", fontSize: 13 }}>{p.nombre}</div>
                  <div style={{ fontSize: 10, color: "#475569" }}>{p.pj} PJ · {p.pts} PTS · {p.eff}% · {p.pg}V {p.pe}E {p.pp}D</div>
                </div>
                <div style={{ display: "flex", gap: 5 }}>
                  <button style={ST.btnBlue} onClick={() => setEditPlayer({ id: p.jugador_id, nombre: p.nombre })}>✏️</button>
                  <button style={ST.btnDanger} onClick={() => deletePlayer(p.jugador_id)}>🗑️</button>
                </div>
              </div>
            ))}
          </div>
        )}

      </div>

      {modal?.type === "edit-match" && (() => {
        const m = partidos.find(x => x.id === modal.id);
        if (!m) return null;
        return (
          <div style={{ position: "fixed", inset: 0, background: "rgba(0,0,0,.88)", display: "flex", alignItems: "center", justifyContent: "center", zIndex: 99, padding: 20 }}>
            <div style={{ background: "#0f172a", border: "1px solid #334155", borderRadius: 14, padding: 22, width: "100%", maxWidth: 380 }}>
              <div style={{ fontWeight: 900, fontSize: 14, marginBottom: 4 }}>✏️ Corregir Resultado</div>
              <div style={{ fontSize: 12, color: "#64748b", marginBottom: 14 }}>Semana {m.semana} · {m.fecha}</div>
              {[["A", "🟢 Ganó Equipo A", "#16a34a"], ["draw", "🟡 Empate", "#ca8a04"], ["B", "🔵 Ganó Equipo B", "#1d4ed8"]].map(([v, l, col]) => (
                <button key={v} onClick={() => fixMatch(m.id, v)}
                  style={{ display: "block", width: "100%", background: m.resultado === v ? col + "22" : "transparent", border: `2px solid ${m.resultado === v ? col : "#334155"}`, color: m.resultado === v ? col : "#94a3b8", borderRadius: 9, padding: "10px 13px", cursor: "pointer", fontWeight: 700, fontSize: 12, textAlign: "left", marginBottom: 7 }}>
                  {l}{m.resultado === v ? " ✓" : ""}
                </button>
              ))}
              <button style={{ ...ST.btnGhost, width: "100%", textAlign: "center" }} onClick={() => setModal(null)}>Cancelar</button>
            </div>
          </div>
        );
      })()}

      {modal?.type === "close-t" && (() => {
        const t = torneos.find(x => x.id === modal.id);
        const tStats = participaciones.filter(p => p.torneo_id === t?.id).map(p => ({ ...p, pts: p.pg * 3 + p.pe, pj: p.pg + p.pe + p.pp })).sort((a, b) => b.pts - a.pts);
        const winner = tStats.find(p => p.pj > 0);
        const winnerNombre = winner ? jugadores.find(j => j.id === winner.jugador_id)?.nombre : null;
        return (
          <div style={{ position: "fixed", inset: 0, background: "rgba(0,0,0,.88)", display: "flex", alignItems: "center", justifyContent: "center", zIndex: 99, padding: 20 }}>
            <div style={{ background: "#0f172a", border: "1px solid #334155", borderRadius: 14, padding: 22, width: "100%", maxWidth: 380 }}>
              <div style={{ fontWeight: 900, fontSize: 15, marginBottom: 6 }}>🏁 Cerrar Torneo</div>
              <div style={{ fontSize: 13, color: "#64748b", marginBottom: 14 }}>{t?.nombre}</div>
              {winnerNombre && (
                <div style={{ background: "#fbbf2418", border: "1px solid #fbbf2433", borderRadius: 9, padding: 12, marginBottom: 14, textAlign: "center" }}>
                  <div style={{ fontSize: 24, marginBottom: 4 }}>🏆</div>
                  <div style={{ fontSize: 15, fontWeight: 900, color: "#fbbf24" }}>CAMPEÓN: {winnerNombre}</div>
                  <div style={{ fontSize: 12, color: "#92400e", marginTop: 3 }}>{winner.pts} puntos · {winner.pg} victorias</div>
                </div>
              )}
              <div style={{ fontSize: 13, color: "#94a3b8", marginBottom: 14 }}>¿Confirmar cierre? Ya no podrás registrar más partidos.</div>
              <div style={{ display: "flex", gap: 7 }}>
                <button style={{ ...ST.btnGhost, flex: 1, textAlign: "center" }} onClick={() => setModal(null)}>Cancelar</button>
                <button style={{ ...ST.btnGreen, flex: 1, padding: 11 }} onClick={() => closeTournament(modal.id)}>🏁 Cerrar</button>
              </div>
            </div>
          </div>
        );
      })()}
    </div>
  );
}