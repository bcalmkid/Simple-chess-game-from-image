<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Spreadsheet Chess</title>
<style>
  :root{
    --grid: 64px;
    --font: 14px;
    --border: #d0d7de;
    --sel: #8ecae64d;
    --hint: #219ebc33;
    --move: #ffb70355;
    --header-bg:#f6f8fa;
  }
  body{font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial, "Apple Color Emoji","Segoe UI Emoji";
       margin:0; color:#111; background:#fff;}
  header{display:flex; gap:12px; align-items:center; padding:12px 16px; border-bottom:1px solid var(--border);}
  header h1{font-size:16px; margin:0 8px 0 0; font-weight:600;}
  button,select{border:1px solid var(--border); background:#fff; padding:6px 10px; border-radius:6px; cursor:pointer;}
  button:hover,select:hover{background:#f7f7f7}
  main{padding:16px; display:grid; grid-template-columns: auto 1fr; gap:18px; align-items:start;}
  .sheet{
    border:1px solid var(--border);
    border-radius:8px;
    overflow:hidden;
    box-shadow:0 1px 0 #00000008 inset;
    background:white;
  }
  .grid{
    display:grid;
    grid-template-columns: 40px repeat(8, var(--grid));
    grid-auto-rows: var(--grid);
    border-top:1px solid var(--border);
    border-left:1px solid var(--border);
    position:relative;
  }
  .hcell{
    background:var(--header-bg);
    font-weight:600;
    font-size:12px;
    color:#57606a;
    display:flex; align-items:center; justify-content:center;
    border-right:1px solid var(--border);
    border-bottom:1px solid var(--border);
  }
  .rhead{
    background:var(--header-bg);
    font-weight:600; font-size:12px; color:#57606a;
    border-right:1px solid var(--border); border-bottom:1px solid var(--border);
    display:flex; align-items:center; justify-content:center;
  }
  .cell{
    border-right:1px solid var(--border);
    border-bottom:1px solid var(--border);
    font-size:var(--font);
    padding:6px;
    display:flex; align-items:center; justify-content:center;
    user-select:none;
    position:relative;
  }
  .cell.white{background:#ffffff}
  .cell.black{background:#fbfdff}
  .cell.sel{background:var(--sel)}
  .cell.hint{box-shadow: inset 0 0 0 9999px var(--hint)}
  .cell.moveTo{box-shadow: inset 0 0 0 9999px var(--move)}
  .piece{display:inline-block; text-align:center; line-height:1.1; padding:4px 6px; border-radius:6px;}
  .w .piece{color:#0b4; font-weight:600}
  .b .piece{color:#333; font-weight:600}
  .panel{display:flex; flex-direction:column; gap:10px;}
  .meta{font-size:13px; color:#444}
  .moves{height: calc(8*var(--grid) + 40px); width:min(320px, 90vw); overflow:auto; border:1px solid var(--border); border-radius:8px; padding:8px;}
  .moves h3{margin:.25rem 0 .5rem 0; font-size:14px}
  .pill{display:inline-block; font-size:12px; padding:2px 6px; border:1px solid var(--border); border-radius:999px; background:#fff; margin-left:8px;}
  .legend{font-size:12px; color:#666}
  footer{padding:10px 16px; color:#666; font-size:12px; border-top:1px solid var(--border);}
  .tiny{font-size:11px; color:#666}
</style>
</head>
<body>
  <header>
    <h1>chess game with</h1>
    <select id="preset">
      <option value="standard">Start: Standard</option>
      <option value="meme">Start: Meme Board (image)</option>
    </select>
    <button id="resetBtn">Reset</button>
    <button id="flipBtn">Flip Board</button>
    <span class="pill" id="turnPill">Turn: White</span>
    <span class="legend">Click a piece, then a destination cell. Words-as-pieces, spreadsheet style.</span>
  </header>

  <main>
    <div class="sheet">
      <div class="grid" id="grid"></div>
    </div>
    <div class="panel">
      <div class="moves" id="moves"><h3>Moves</h3><ol id="movelist"></ol></div>
      <div class="meta">
        <strong>Notes</strong>
        <ul>
          <li>Enforces how pieces move & captures; includes pawn double-step and auto-promotion to queen.</li>
          <li>No checks/checkmate detection, castling, or en-passant (kept compact).</li>
          <li>Use <em>Flip Board</em> to view from Black’s side.</li>
        </ul>
      </div>
      <div class="tiny">Made for easy self-hosting — one HTML file, no dependencies.</div>
    </div>
  </main>

  <footer>
    Based on the spreadsheet-style board from your image (with words instead of icons).
  </footer>

<script>
/* ---------- Model ---------- */
const FILE = {
  cols: ['A','B','C','D','E','F','G','H'],
  rows: [8,7,6,5,4,3,2,1],   // top to bottom
};

const INITIALS = {
  standard: () => ({
    // white bottom
    A1:'w rook', B1:'w knight', C1:'w bishop', D1:'w queen', E1:'w king', F1:'w bishop', G1:'w knight', H1:'w rook',
    A2:'w pawn', B2:'w pawn', C2:'w pawn', D2:'w pawn', E2:'w pawn', F2:'w pawn', G2:'w pawn', H2:'w pawn',
    A7:'b pawn', B7:'b pawn', C7:'b pawn', D7:'b pawn', E7:'b pawn', F7:'b pawn', G7:'b pawn', H7:'b pawn',
    A8:'b rook', B8:'b knight', C8:'b bishop', D8:'b queen', E8:'b king', F8:'b bishop', G8:'b knight', H8:'b rook',
  }),
  meme: () => {
    const b = INITIALS.standard();
    // replicate the image: keep normal start AND add an extra white pawn on E5
    b.E5 = 'w pawn';
    return b;
  }
};

// Board state
let board = {};
let whiteToMove = true;
let selected = null;
let flip = false;
const gridEl = document.getElementById('grid');
const movesEl = document.getElementById('movelist');
const turnPill = document.getElementById('turnPill');

/* ---------- Helpers ---------- */
const coord = (c,r) => `${c}${r}`;
const inside = (c,r) => FILE.cols.includes(c) && FILE.rows.includes(r);
const parse = sq => [sq[0], parseInt(sq.slice(1),10)];
const opp = side => side === 'w' ? 'b' : 'w';
const clone = obj => JSON.parse(JSON.stringify(obj));

function pieceAt(sq){ return board[sq] || null; }
function colorOf(piece){ return piece ? piece.split(' ')[0] : null; }
function typeOf(piece){ return piece ? piece.split(' ')[1] : null; }

function setTurnPill(){
  turnPill.textContent = `Turn: ${whiteToMove ? 'White':'Black'}`;
  turnPill.style.background = whiteToMove ? '#eefcf1' : '#eef2ff';
}

/* ---------- Move Generation (pseudo-legal) ---------- */
function ray(c, r, dc, dr, side){
  const out = [];
  while(true){
    const ci = FILE.cols.indexOf(c) + dc;
    const ri = FILE.rows.indexOf(r) + dr;
    if(ci<0||ci>7||ri<0||ri>7) break;
    c = FILE.cols[ci]; r = FILE.rows[ri];
    const sq = coord(c,r);
    const p = pieceAt(sq);
    if(!p){ out.push(sq); continue; }
    if(colorOf(p) !== side){ out.push(sq); }
    break;
  }
  return out;
}

function pawnMoves(sq, side){
  const [c,r] = parse(sq);
  const dir = side==='w' ? 1 : -1; // because rows array is 8..1 top->bottom
  const rIdx = FILE.rows.indexOf(r);
  const oneIdx = rIdx + (side==='w' ? -1 : +1); // move visually “down” for white
  const twoIdx = rIdx + (side==='w' ? -2 : +2);
  const out = [];
  // forward
  if(oneIdx>=0 && oneIdx<8){
    const fwd = coord(c, FILE.rows[oneIdx]);
    if(!pieceAt(fwd)){
      out.push(fwd);
      const startRank = side==='w' ? 2 : 7;
      if(r===startRank && twoIdx>=0 && twoIdx<8){
        const two = coord(c, FILE.rows[twoIdx]);
        if(!pieceAt(two)) out.push(two);
      }
    }
  }
  // captures
  const colIdx = FILE.cols.indexOf(c);
  const diagRowIdx = oneIdx;
  if(diagRowIdx>=0 && diagRowIdx<8){
    for(const dc of [-1,1]){
      const ci = colIdx + dc;
      if(ci<0||ci>7) continue;
      const cs = FILE.cols[ci];
      const sqd = coord(cs, FILE.rows[diagRowIdx]);
      const p = pieceAt(sqd);
      if(p && colorOf(p)!==side) out.push(sqd);
    }
  }
  return out;
}

function knightMoves(sq){
  const [c,r] = parse(sq); const ci = FILE.cols.indexOf(c); const ri = FILE.rows.indexOf(r);
  const deltas = [[1,2],[2,1],[-1,2],[-2,1],[1,-2],[2,-1],[-1,-2],[-2,-1]];
  const out=[];
  for(const [dx,dy] of deltas){
    const x=ci+dx, y=ri+dy;
    if(x<0||x>7||y<0||y>7) continue;
    const t = coord(FILE.cols[x], FILE.rows[y]);
    out.push(t);
  }
  return out;
}

function kingMoves(sq){ // no castling
  const [c,r]=parse(sq); const ci=FILE.cols.indexOf(c); const ri=FILE.rows.indexOf(r);
  const out=[];
  for(let dx=-1; dx<=1; dx++){
    for(let dy=-1; dy<=1; dy++){
      if(dx===0&&dy===0) continue;
      const x=ci+dx, y=ri+dy;
      if(x<0||x>7||y<0||y>7) continue;
      out.push(coord(FILE.cols[x], FILE.rows[y]));
    }
  }
  return out;
}

function legalDestinations(sq){
  const p = pieceAt(sq); if(!p) return [];
  const side = colorOf(p); const type = typeOf(p);
  let pseudo=[];
  if(type==='pawn') pseudo = pawnMoves(sq, side);
  else if(type==='knight') pseudo = knightMoves(sq);
  else if(type==='king') pseudo = kingMoves(sq);
  else if(type==='rook'){
    pseudo=[...ray(...parse(sq), +1,0, side), ...ray(...parse(sq), -1,0, side),
            ...ray(...parse(sq), 0,+1, side), ...ray(...parse(sq), 0,-1, side)];
  } else if(type==='bishop'){
    pseudo=[...ray(...parse(sq), +1,+1, side), ...ray(...parse(sq), +1,-1, side),
            ...ray(...parse(sq), -1,+1, side), ...ray(...parse(sq), -1,-1, side)];
  } else if(type==='queen'){
    pseudo=[...legalDestinations(sq.replace(type,'rook')),
            ...legalDestinations(sq.replace(type,'bishop'))]; // quick reuse
  }
  // filter out own-occupied squares
  return pseudo.filter(t => {
    const occ = pieceAt(t);
    return !occ || colorOf(occ)!==side;
  });
}

/* ---------- Rendering ---------- */
function buildGrid(){
  gridEl.innerHTML = '';
  // top-left empty header
  const tl = document.createElement('div'); tl.className='hcell'; tl.textContent='';
  gridEl.appendChild(tl);
  // column headers
  const cols = flip ? [...FILE.cols].reverse() : FILE.cols;
  const rows = flip ? [...FILE.rows].reverse() : FILE.rows;
  cols.forEach(c=>{
    const h=document.createElement('div'); h.className='hcell'; h.textContent=c; gridEl.appendChild(h);
  });
  // rows
  for(const r of rows){
    const rh = document.createElement('div'); rh.className='rhead'; rh.textContent=r; gridEl.appendChild(rh);
    for(const c of cols){
      const sq = coord(c,r);
      const cell = document.createElement('div');
      cell.className = 'cell ' + (((FILE.cols.indexOf(c)+FILE.rows.indexOf(r))%2===0)?'white':'black');
      cell.dataset.sq = sq;
      const p = pieceAt(sq);
      if(p){
        const side = colorOf(p);
        const t = typeOf(p);
        const span = document.createElement('span');
        span.className = `piece ${side==='w'?'w':'b'}`;
        span.textContent = t;
        cell.appendChild(span);
        cell.classList.add(side==='w'?'w':'b');
      }
      cell.addEventListener('click', onCellClick);
      gridEl.appendChild(cell);
    }
  }
}

function clearHints(){
  gridEl.querySelectorAll('.cell').forEach(c=>{
    c.classList.remove('sel','hint','moveTo');
  });
}

function showHints(from, targets){
  clearHints();
  const cFrom = gridEl.querySelector(`.cell[data-sq="${from}"]`);
  if(cFrom) cFrom.classList.add('sel');
  for(const t of targets){
    const el = gridEl.querySelector(`.cell[data-sq="${t}"]`);
    if(!el) continue;
    if(el.children.length) el.classList.add('moveTo'); else el.classList.add('hint');
  }
}

/* ---------- Interaction ---------- */
function onCellClick(e){
  const sq = e.currentTarget.dataset.sq;
  const p = pieceAt(sq);
  if(selected){
    // If second click is same color piece: reselect
    if(p && colorOf(p) === (whiteToMove?'w':'b')){
      selected = sq;
      showHints(sq, legalDestinations(sq).filter(t=>canMove(selected,t)));
      return;
    }
    // attempt move
    const ok = attemptMove(selected, sq);
    if(ok){
      selected = null;
      clearHints();
      buildGrid();
      setTurnPill();
      return;
    }
    // invalid -> keep selection
    return;
  }
  // no selection yet
  if(p && colorOf(p) === (whiteToMove?'w':'b')){
    selected = sq;
    showHints(sq, legalDestinations(sq).filter(t=>canMove(selected,t)));
  }
}

function attemptMove(from, to){
  const legals = legalDestinations(from);
  if(!legals.includes(to)) return false;
  // execute
  const p = board[from];
  delete board[from];
  // promotion
  if(typeOf(p)==='pawn' && (to.endsWith('8') || to.endsWith('1'))){
    board[to] = colorOf(p) + ' queen';
  } else {
    board[to] = p;
  }
  addMoveRecord(from, to, p, board[to]!==p);
  whiteToMove = !whiteToMove;
  return true;
}

function canMove(from,to){
  // simple occupancy rule already handled; no check detection
  return true;
}

function addMoveRecord(from, to, piece, capture){
  const li = document.createElement('li');
  const side = colorOf(piece)==='w'?'White':'Black';
  const name = typeOf(piece);
  li.textContent = `${side}: ${name} ${from} → ${to}` + (capture?' (x)':'');
  movesEl.appendChild(li);
  // keep scrolled
  const movesBox = document.getElementById('moves');
  movesBox.scrollTop = movesBox.scrollHeight;
}

/* ---------- Controls ---------- */
function reset(presetName){
  board = INITIALS[presetName]();
  whiteToMove = true;
  selected = null;
  clearHints();
  movesEl.innerHTML = '';
  buildGrid();
  setTurnPill();
}

document.getElementById('resetBtn').addEventListener('click', ()=>{
  reset(document.getElementById('preset').value);
});
document.getElementById('flipBtn').addEventListener('click', ()=>{
  flip = !flip; buildGrid();
});

/* ---------- Boot ---------- */
reset('standard'); // default
</script>
</body>
</html>
