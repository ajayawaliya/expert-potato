# expert-potato
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Ultimate Arcade</title>
<style>
body { margin:0; overflow:hidden; background:black; font-family:Arial; }
canvas { display:block; }
#ui {
  position:absolute;
  top:10px;
  left:10px;
  color:white;
}
button {
  padding:8px;
  margin:4px;
}
</style>
</head>
<body>
<canvas id="game"></canvas>
<div id="ui"></div>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

/* =======================
   STATE SYSTEM
======================= */
let state = "menu";
let score = 0;
let level = 1;
let coins = 0;
let highScore = localStorage.getItem("highScore") || 0;
let achievements = JSON.parse(localStorage.getItem("achievements")||"[]");

/* =======================
   INPUT
======================= */
let keys = {};
document.addEventListener("keydown", e=>keys[e.code]=true);
document.addEventListener("keyup", e=>keys[e.code]=false);

let touchX=null;
canvas.addEventListener("touchmove",e=>{
  touchX=e.touches[0].clientX;
});
canvas.addEventListener("touchstart",()=>shoot());

/* =======================
   GAMEPAD
======================= */
let gamepadIndex=null;
window.addEventListener("gamepadconnected",(e)=>{
  gamepadIndex=e.gamepad.index;
});

function handleGamepad(){
  if(gamepadIndex===null) return;
  let gp=navigator.getGamepads()[gamepadIndex];
  if(gp.axes[0]<-0.2) player.x-=player.speed;
  if(gp.axes[0]>0.2) player.x+=player.speed;
  if(gp.buttons[0].pressed) shoot();
}

/* =======================
   SOUND ENGINE
======================= */
const laserSound=new Audio();
const boomSound=new Audio();

/* =======================
   PLAYER
======================= */
let player={
  x:canvas.width/2-25,
  y:canvas.height-100,
  width:50,
  height:50,
  speed:7,
  health:100,
  lives:3,
  cooldown:0
};

/* =======================
   PARTICLES
======================= */
let particles=[];
function explosion(x,y){
  for(let i=0;i<30;i++){
    particles.push({
      x,y,
      vx:(Math.random()-0.5)*6,
      vy:(Math.random()-0.5)*6,
      life:50
    });
  }
}

function drawParticles(){
  particles.forEach((p,i)=>{
    ctx.fillStyle="orange";
    ctx.fillRect(p.x,p.y,3,3);
    p.x+=p.vx;
    p.y+=p.vy;
    p.life--;
    if(p.life<=0) particles.splice(i,1);
  });
}

/* =======================
   NEBULA BACKGROUND
======================= */
let stars=[];
for(let i=0;i<120;i++){
  stars.push({
    x:Math.random()*canvas.width,
    y:Math.random()*canvas.height,
    size:Math.random()*2,
    speed:Math.random()*0.5
  });
}

function drawBackground(){
  let g=ctx.createRadialGradient(
    canvas.width/2,canvas.height/2,100,
    canvas.width/2,canvas.height/2,canvas.width
  );
  g.addColorStop(0,"#1a0033");
  g.addColorStop(1,"black");
  ctx.fillStyle=g;
  ctx.fillRect(0,0,canvas.width,canvas.height);

  ctx.fillStyle="white";
  stars.forEach(s=>{
    ctx.fillRect(s.x,s.y,s.size,s.size);
    s.y+=s.speed;
    if(s.y>canvas.height) s.y=0;
  });
}

/* =======================
   BULLETS
======================= */
let bullets=[];
function shoot(){
  if(player.cooldown>0) return;
  bullets.push({
    x:player.x+20,
    y:player.y,
    speed:10
  });
  player.cooldown=15;
}

function drawBullets(){
  ctx.fillStyle="yellow";
  bullets.forEach((b,i)=>{
    ctx.fillRect(b.x,b.y,5,15);
    b.y-=b.speed;
    if(b.y<0) bullets.splice(i,1);
  });
}

/* =======================
   ENEMIES + AI
======================= */
let enemies=[];
function spawnEnemy(){
  enemies.push({
    x:Math.random()*canvas.width,
    y:-40,
    speed:2+level*0.2
  });
}

function drawEnemies(){
  ctx.fillStyle="red";
  enemies.forEach((e,ei)=>{
    // adaptive difficulty
    e.y+=e.speed + score/1000;
    if(e.x<player.x) e.x+=1;
    if(e.x>player.x) e.x-=1;

    ctx.fillRect(e.x,e.y,40,40);

    // collision
    bullets.forEach((b,bi)=>{
      if(b.x<e.x+40 && b.x+5>e.x &&
         b.y<e.y+40 && b.y+15>e.y){
        explosion(e.x,e.y);
        enemies.splice(ei,1);
        bullets.splice(bi,1);
        score+=10;
        coins+=1;
      }
    });

    if(e.y>canvas.height){
      player.health-=10;
      enemies.splice(ei,1);
    }
  });
}

/* =======================
   ACHIEVEMENTS
======================= */
function unlock(name){
  if(!achievements.includes(name)){
    achievements.push(name);
    localStorage.setItem("achievements",JSON.stringify(achievements));
    alert("🏅 "+name);
  }
}

/* =======================
   UPDATE
======================= */
function update(){
  if(state!=="playing") return;

  if(keys["ArrowLeft"]) player.x-=player.speed;
  if(keys["ArrowRight"]) player.x+=player.speed;
  if(keys["Space"]) shoot();

  if(touchX!==null) player.x=touchX-25;

  handleGamepad();

  if(player.cooldown>0) player.cooldown--;

  if(score>500) unlock("Score 500!");
  if(score>1000) unlock("Score 1000!");
}

/* =======================
   DRAW UI
======================= */
function drawUI(){
  ctx.fillStyle="white";
  ctx.fillText("Score: "+score,20,30);
  ctx.fillText("High: "+highScore,20,50);
  ctx.fillText("Coins: "+coins,20,70);

  ctx.fillStyle="red";
  ctx.fillRect(20,90,100,10);
  ctx.fillStyle="lime";
  ctx.fillRect(20,90,player.health,10);
}

/* =======================
   LOOP
======================= */
function loop(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  drawBackground();
  update();
  drawBullets();
  drawEnemies();
  drawParticles();

  ctx.fillStyle="blue";
  ctx.fillRect(player.x,player.y,50,50);

  drawUI();

  requestAnimationFrame(loop);
}

setInterval(()=>{ if(state==="playing") spawnEnemy(); },1000);

document.addEventListener("keydown",e=>{
  if(e.code==="Enter"){
    state="playing";
  }
});

loop();
</script>
</body>
</html>
