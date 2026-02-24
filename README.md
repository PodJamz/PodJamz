import { useState, useMemo, useEffect } from "react";

// Generate seeded pseudo-random contribution data
function seededRandom(seed) {
  let s = seed;
  return () => {
    s = (s * 16807 + 0) % 2147483647;
    return (s - 1) / 2147483646;
  };
}

function generateContributions(weeks = 52) {
  const rng = seededRandom(42);
  const data = [];
  const now = new Date();
  const start = new Date(now);
  start.setDate(start.getDate() - weeks * 7);

  for (let w = 0; w < weeks; w++) {
    const week = [];
    for (let d = 0; d < 7; d++) {
      const date = new Date(start);
      date.setDate(date.getDate() + w * 7 + d);
      const r = rng();
      // Weighted distribution: more zeros, some bursts
      let count = 0;
      if (r > 0.55) count = Math.floor(r * 4) + 1;
      if (r > 0.85) count = Math.floor(r * 12) + 4;
      if (r > 0.95) count = Math.floor(r * 20) + 8;
      // Weekend reduction
      if (d === 0 || d === 6) count = Math.floor(count * 0.4);
      week.push({ date, count });
    }
    data.push(week);
  }
  return data;
}

const MONTHS = ["JAN", "FEB", "MAR", "APR", "MAY", "JUN", "JUL", "AUG", "SEP", "OCT", "NOV", "DEC"];

function getLevel(count) {
  if (count === 0) return 0;
  if (count <= 3) return 1;
  if (count <= 6) return 2;
  if (count <= 10) return 3;
  return 4;
}

// TE-style colors
const COLORS = {
  bg: "#1a1a1a",
  deviceBody: "#2a2a2a",
  deviceEdge: "#1f1f1f",
  deviceBottom: "#151515",
  screen: "#0d0d0d",
  screenBorder: "#333",
  orange: "#FF6B00",
  orangeDim: "#CC5500",
  orangeGlow: "#FF8C33",
  text: "#888",
  textBright: "#ccc",
  label: "#555",
  knob: "#3a3a3a",
  knobEdge: "#2a2a2a",
  led: "#FF6B00",
  ledOff: "#331a00",
  port: "#111",
  levels: ["#161616", "#FF6B0022", "#FF6B0055", "#FF6B00aa", "#FF6B00"],
};

function Knob({ size = 28, label, value, style }) {
  const rotation = (value || 0) * 270 - 135;
  return (
    <div style={{ display: "flex", flexDirection: "column", alignItems: "center", gap: 3, ...style }}>
      <div
        style={{
          width: size,
          height: size,
          borderRadius: "50%",
          background: `radial-gradient(circle at 40% 35%, #4a4a4a, ${COLORS.knob} 60%, ${COLORS.knobEdge})`,
          boxShadow: "0 2px 4px rgba(0,0,0,0.5), inset 0 1px 1px rgba(255,255,255,0.05)",
          position: "relative",
          cursor: "pointer",
          border: "1px solid #222",
        }}
      >
        <div
          style={{
            position: "absolute",
            top: "15%",
            left: "50%",
            width: 2,
            height: "30%",
            background: COLORS.orange,
            borderRadius: 1,
            transformOrigin: "bottom center",
            transform: `translateX(-50%) rotate(${rotation}deg)`,
          }}
        />
      </div>
      {label && (
        <span style={{ fontSize: 7, fontFamily: "'Space Mono', monospace", color: COLORS.label, letterSpacing: 1.5, textTransform: "uppercase" }}>
          {label}
        </span>
      )}
    </div>
  );
}

function LED({ on = true, size = 6 }) {
  return (
    <div
      style={{
        width: size,
        height: size,
        borderRadius: "50%",
        background: on ? COLORS.led : COLORS.ledOff,
        boxShadow: on ? `0 0 6px ${COLORS.orange}, 0 0 12px ${COLORS.orange}44` : "none",
        border: "1px solid #222",
      }}
    />
  );
}

function Port({ width = 14, height = 8 }) {
  return (
    <div
      style={{
        width,
        height,
        borderRadius: 2,
        background: COLORS.port,
        border: "1px solid #0a0a0a",
        boxShadow: "inset 0 1px 3px rgba(0,0,0,0.8)",
      }}
    />
  );
}

export default function TEActivityGraph() {
  const [hoveredCell, setHoveredCell] = useState(null);
  const [rotateX, setRotateX] = useState(8);
  const [rotateY, setRotateY] = useState(-4);
  const [isDragging, setIsDragging] = useState(false);
  const [dragStart, setDragStart] = useState({ x: 0, y: 0, rx: 0, ry: 0 });
  const [viewMode, setViewMode] = useState(0); // 0=year, 1=6mo, 2=3mo
  const [screenOn, setScreenOn] = useState(true);
  const [breathe, setBreathe] = useState(0);

  const contributions = useMemo(() => generateContributions(52), []);

  const weeksToShow = viewMode === 0 ? 52 : viewMode === 1 ? 26 : 13;
  const visibleWeeks = contributions.slice(-weeksToShow);

  const totalContribs = visibleWeeks.flat().reduce((sum, d) => sum + d.count, 0);
  const maxDay = visibleWeeks.flat().reduce((max, d) => (d.count > max.count ? d : max), { count: 0 });

  // Breathing LED animation
  useEffect(() => {
    const interval = setInterval(() => {
      setBreathe((b) => (b + 0.05) % (Math.PI * 2));
    }, 50);
    return () => clearInterval(interval);
  }, []);

  const handleMouseDown = (e) => {
    if (e.target.closest("[data-interactive]")) return;
    setIsDragging(true);
    setDragStart({ x: e.clientX, y: e.clientY, rx: rotateX, ry: rotateY });
  };

  const handleMouseMove = (e) => {
    if (!isDragging) return;
    const dx = e.clientX - dragStart.x;
    const dy = e.clientY - dragStart.y;
    setRotateX(Math.max(-20, Math.min(25, dragStart.rx - dy * 0.3)));
    setRotateY(Math.max(-25, Math.min(25, dragStart.ry + dx * 0.3)));
  };

  const handleMouseUp = () => setIsDragging(false);

  const cellSize = 11;
  const cellGap = 2;
  const gridW = visibleWeeks.length * (cellSize + cellGap);
  const gridH = 7 * (cellSize + cellGap);

  // Month labels
  const monthLabels = [];
  let lastMonth = -1;
  visibleWeeks.forEach((week, i) => {
    const month = week[0].date.getMonth();
    if (month !== lastMonth) {
      monthLabels.push({ label: MONTHS[month], x: i * (cellSize + cellGap) });
      lastMonth = month;
    }
  });

  const deviceW = Math.max(gridW + 80, 520);
  const deviceH = 280;
  const depth = 16;

  return (
    <div
      style={{
        width: "100%",
        minHeight: "100vh",
        display: "flex",
        alignItems: "center",
        justifyContent: "center",
        background: COLORS.bg,
        fontFamily: "'Space Mono', 'SF Mono', 'Fira Code', monospace",
        perspective: 1200,
        cursor: isDragging ? "grabbing" : "grab",
        userSelect: "none",
        overflow: "hidden",
      }}
      onMouseMove={handleMouseMove}
      onMouseUp={handleMouseUp}
      onMouseLeave={handleMouseUp}
    >
      <link href="https://fonts.googleapis.com/css2?family=Space+Mono:wght@400;700&display=swap" rel="stylesheet" />

      <div
        onMouseDown={handleMouseDown}
        style={{
          transformStyle: "preserve-3d",
          transform: `rotateX(${rotateX}deg) rotateY(${rotateY}deg)`,
          transition: isDragging ? "none" : "transform 0.3s ease-out",
        }}
      >
        {/* === DEVICE TOP FACE === */}
        <div
          style={{
            width: deviceW,
            height: deviceH,
            background: `linear-gradient(180deg, #333 0%, ${COLORS.deviceBody} 3%, ${COLORS.deviceBody} 97%, #222 100%)`,
            borderRadius: 10,
            position: "relative",
            transformStyle: "preserve-3d",
            boxShadow: "0 -1px 0 #444 inset, 0 30px 60px rgba(0,0,0,0.5)",
            border: "1px solid #3a3a3a",
            padding: 16,
            display: "flex",
            flexDirection: "column",
          }}
        >
          {/* Top bar - label + knobs */}
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 8, height: 36 }}>
            <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
              <div style={{ display: "flex", alignItems: "center", gap: 6 }}>
                <LED on={screenOn} size={5} />
                <span
                  style={{
                    fontSize: 9,
                    fontWeight: 700,
                    color: COLORS.orange,
                    letterSpacing: 3,
                    textTransform: "uppercase",
                  }}
                >
                  GH-01
                </span>
              </div>
              <span style={{ fontSize: 7, color: COLORS.label, letterSpacing: 2 }}>CONTRIBUTION ENGINE</span>
            </div>

            <div style={{ display: "flex", alignItems: "center", gap: 12 }} data-interactive="true">
              <Knob size={24} label="ZOOM" value={viewMode / 2} />
              <Knob size={24} label="GAIN" value={0.6} />
              <div
                onClick={() => setScreenOn(!screenOn)}
                style={{
                  width: 14,
                  height: 14,
                  borderRadius: 3,
                  background: screenOn ? COLORS.orange : "#333",
                  border: "1px solid #222",
                  cursor: "pointer",
                  boxShadow: screenOn ? `0 0 8px ${COLORS.orange}66` : "inset 0 1px 2px rgba(0,0,0,0.5)",
                  transition: "all 0.2s",
                }}
              />
            </div>
          </div>

          {/* Screen area */}
          <div
            style={{
              flex: 1,
              background: screenOn ? COLORS.screen : "#080808",
              borderRadius: 4,
              border: `1px solid ${COLORS.screenBorder}`,
              padding: "10px 14px",
              position: "relative",
              overflow: "hidden",
              boxShadow: screenOn ? `inset 0 0 30px rgba(255,107,0,0.03)` : "none",
              transition: "all 0.4s",
            }}
          >
            {screenOn && (
              <>
                {/* Scanline effect */}
                <div
                  style={{
                    position: "absolute",
                    inset: 0,
                    background: "repeating-linear-gradient(0deg, transparent, transparent 2px, rgba(0,0,0,0.08) 2px, rgba(0,0,0,0.08) 4px)",
                    pointerEvents: "none",
                    zIndex: 10,
                  }}
                />

                {/* Month labels */}
                <div style={{ display: "flex", marginBottom: 4, marginLeft: 20, height: 12, position: "relative" }}>
                  {monthLabels.map((m, i) => (
                    <span
                      key={i}
                      style={{
                        position: "absolute",
                        left: m.x,
                        fontSize: 7,
                        color: COLORS.label,
                        letterSpacing: 1.5,
                      }}
                    >
                      {m.label}
                    </span>
                  ))}
                </div>

                {/* Day labels + Grid */}
                <div style={{ display: "flex", gap: 0 }}>
                  <div style={{ display: "flex", flexDirection: "column", gap: cellGap, marginRight: 4, paddingTop: 0 }}>
                    {["M", "", "W", "", "F", "", ""].map((d, i) => (
                      <div key={i} style={{ height: cellSize, display: "flex", alignItems: "center" }}>
                        <span style={{ fontSize: 7, color: COLORS.label, width: 14, textAlign: "right" }}>{d}</span>
                      </div>
                    ))}
                  </div>

                  <div style={{ display: "flex", gap: cellGap }}>
                    {visibleWeeks.map((week, wi) => (
                      <div key={wi} style={{ display: "flex", flexDirection: "column", gap: cellGap }}>
                        {week.map((day, di) => {
                          const level = getLevel(day.count);
                          const isHovered = hoveredCell?.w === wi && hoveredCell?.d === di;
                          return (
                            <div
                              key={di}
                              onMouseEnter={() => setHoveredCell({ w: wi, d: di, ...day })}
                              onMouseLeave={() => setHoveredCell(null)}
                              style={{
                                width: cellSize,
                                height: cellSize,
                                borderRadius: 2,
                                background: COLORS.levels[level],
                                border: isHovered ? `1px solid ${COLORS.orange}` : "1px solid #1a1a1a",
                                cursor: "crosshair",
                                transition: "all 0.15s",
                                transform: isHovered ? "scale(1.4)" : "scale(1)",
                                boxShadow: isHovered ? `0 0 8px ${COLORS.orange}44` : level >= 3 ? `0 0 4px ${COLORS.orange}22` : "none",
                                zIndex: isHovered ? 20 : 1,
                                position: "relative",
                              }}
                            />
                          );
                        })}
                      </div>
                    ))}
                  </div>
                </div>

                {/* Tooltip */}
                {hoveredCell && (
                  <div
                    style={{
                      position: "absolute",
                      bottom: 8,
                      right: 12,
                      background: "#111",
                      border: `1px solid ${COLORS.orange}44`,
                      borderRadius: 3,
                      padding: "4px 8px",
                      fontSize: 8,
                      color: COLORS.textBright,
                      letterSpacing: 1,
                      zIndex: 30,
                      display: "flex",
                      gap: 8,
                    }}
                  >
                    <span style={{ color: COLORS.orange }}>{hoveredCell.count}</span>
                    <span style={{ color: COLORS.label }}>
                      {hoveredCell.date.toLocaleDateString("en-US", { month: "short", day: "numeric", year: "numeric" })}
                    </span>
                  </div>
                )}
              </>
            )}
          </div>

          {/* Bottom bar - stats + ports */}
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginTop: 8, height: 28 }}>
            <div style={{ display: "flex", alignItems: "center", gap: 16 }}>
              <div style={{ display: "flex", flexDirection: "column", gap: 1 }}>
                <span style={{ fontSize: 7, color: COLORS.label, letterSpacing: 1.5 }}>TOTAL</span>
                <span style={{ fontSize: 11, color: screenOn ? COLORS.orange : COLORS.label, fontWeight: 700, letterSpacing: 1 }}>
                  {screenOn ? totalContribs.toLocaleString() : "---"}
                </span>
              </div>
              <div style={{ width: 1, height: 20, background: "#333" }} />
              <div style={{ display: "flex", flexDirection: "column", gap: 1 }}>
                <span style={{ fontSize: 7, color: COLORS.label, letterSpacing: 1.5 }}>PEAK</span>
                <span style={{ fontSize: 11, color: screenOn ? COLORS.orangeGlow : COLORS.label, fontWeight: 700, letterSpacing: 1 }}>
                  {screenOn ? maxDay.count : "--"}
                </span>
              </div>
              <div style={{ width: 1, height: 20, background: "#333" }} />
              <div data-interactive="true" style={{ display: "flex", gap: 4, alignItems: "center" }}>
                {[0, 1, 2].map((m) => (
                  <div
                    key={m}
                    onClick={() => setViewMode(m)}
                    style={{
                      padding: "2px 6px",
                      fontSize: 7,
                      letterSpacing: 1.5,
                      borderRadius: 2,
                      cursor: "pointer",
                      color: viewMode === m ? COLORS.orange : COLORS.label,
                      background: viewMode === m ? "#FF6B0018" : "transparent",
                      border: viewMode === m ? `1px solid ${COLORS.orange}44` : "1px solid transparent",
                      transition: "all 0.2s",
                    }}
                  >
                    {["1Y", "6M", "3M"][m]}
                  </div>
                ))}
              </div>
            </div>

            <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
              <Port width={10} height={6} />
              <Port width={16} height={8} />
              <LED on={screenOn} size={4} />
              <span style={{ fontSize: 6, color: COLORS.label, letterSpacing: 2 }}>USB-C</span>
            </div>
          </div>

          {/* === RIGHT EDGE (3D) === */}
          <div
            style={{
              position: "absolute",
              top: 0,
              right: -depth,
              width: depth,
              height: deviceH,
              background: `linear-gradient(90deg, ${COLORS.deviceEdge}, #181818)`,
              transformOrigin: "left center",
              transform: "rotateY(90deg)",
              borderRadius: "0 4px 4px 0",
            }}
          >
            {/* Side ports/vents */}
            <div style={{ position: "absolute", top: "30%", left: 3, display: "flex", flexDirection: "column", gap: 3 }}>
              {[0, 1, 2, 3, 4].map((i) => (
                <div key={i} style={{ width: depth - 6, height: 1.5, background: "#111", borderRadius: 1 }} />
              ))}
            </div>
          </div>

          {/* === LEFT EDGE (3D) === */}
          <div
            style={{
              position: "absolute",
              top: 0,
              left: -depth,
              width: depth,
              height: deviceH,
              background: `linear-gradient(270deg, ${COLORS.deviceEdge}, #181818)`,
              transformOrigin: "right center",
              transform: "rotateY(-90deg)",
              borderRadius: "4px 0 0 4px",
            }}
          >
            <div style={{ position: "absolute", top: "50%", left: 3, transform: "translateY(-50%)" }}>
              <div style={{ width: depth - 6, height: 6, background: "#111", borderRadius: 1, border: "1px solid #0a0a0a" }} />
            </div>
          </div>

          {/* === BOTTOM EDGE (3D) === */}
          <div
            style={{
              position: "absolute",
              bottom: -depth,
              left: 0,
              width: deviceW,
              height: depth,
              background: `linear-gradient(180deg, ${COLORS.deviceEdge}, ${COLORS.deviceBottom})`,
              transformOrigin: "top center",
              transform: "rotateX(90deg)",
              borderRadius: "0 0 4px 4px",
            }}
          />

          {/* === TOP EDGE (3D) === */}
          <div
            style={{
              position: "absolute",
              top: -depth,
              left: 0,
              width: deviceW,
              height: depth,
              background: `linear-gradient(0deg, ${COLORS.deviceEdge}, #222)`,
              transformOrigin: "bottom center",
              transform: "rotateX(-90deg)",
              borderRadius: "4px 4px 0 0",
            }}
          />

          {/* Engraved brand text on body */}
          <div
            style={{
              position: "absolute",
              bottom: -2,
              right: 20,
              fontSize: 7,
              color: "#333",
              letterSpacing: 4,
              transform: "translateY(-100%)",
              fontWeight: 700,
            }}
          >
            PODJAMZ™
          </div>
        </div>

        {/* Shadow on ground plane */}
        <div
          style={{
            position: "absolute",
            bottom: -40,
            left: "5%",
            width: "90%",
            height: 30,
            background: "radial-gradient(ellipse, rgba(0,0,0,0.4) 0%, transparent 70%)",
            filter: "blur(10px)",
            transform: "rotateX(90deg)",
            transformOrigin: "top center",
          }}
        />
      </div>

      {/* Instructions */}
      <div
        style={{
          position: "fixed",
          bottom: 20,
          left: "50%",
          transform: "translateX(-50%)",
          fontSize: 9,
          color: COLORS.label,
          letterSpacing: 2,
          display: "flex",
          gap: 16,
          alignItems: "center",
        }}
      >
        <span>DRAG TO ROTATE</span>
        <span style={{ color: "#333" }}>|</span>
        <span>HOVER CELLS</span>
        <span style={{ color: "#333" }}>|</span>
        <span style={{ color: COLORS.orange }}>GH-01</span>
      </div>
    </div>
  );
}
