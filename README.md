# Plants vs Zombies

## Overview
Plants vs Zombies is a popular tower defense video game where players strategically place various plant characters to protect their home from an onslaught of zombies. Players must plan their defenses wisely, as each plant type has unique abilities, and different zombies have different strengths and weaknesses.

## Installation Instructions
To install Plants vs Zombies, follow these steps:
1. Download the installation file from the official website or your preferred platform.
2. Run the installer and follow the on-screen instructions to complete the installation.
3. Once installed, launch the game from your desktop or applications folder.

## Gameplay Guide
- **Objective:** Prevent zombies from reaching your house by strategically placing plants in your garden.
- **Plant Types:** Different plants provide various abilities such as shooting projectiles, blocking zombies, or producing sunlight.
- **Zombies:** Different types of zombies will attack; some can jump over plants, while others fly or break barriers. 
- **Levels:** The game consists of multiple levels with increasing difficulty, introducing new plant and zombie types. 

## Deployment Information
Plant vs Zombies is available for deployment on multiple platforms, including PC, macOS, and mobile devices. You can also play online via gaming platforms that host the game.

For any issues, consult the official guide or community forums for assistance.
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, minimum-scale=1.0, viewport-fit=cover">
  <title>🌱 植物大战僵尸 · 像素重制版</title>
  <!-- 导入像素字体 -->
  <link href="https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap" rel="stylesheet">
  <style>
    * { user-select:none; -webkit-tap-highlight-color:transparent; box-sizing:border-box; }
    body { margin:0; min-height:100vh; background: radial-gradient(circle at 20% 30%, #1c4d2d, #0a2a18); display:flex; justify-content:center; align-items:center; font-family:'Press Start 2P', monospace; padding:16px; touch-action:none; }
    .game-wrapper { background:#2f4d2d; border-radius:48px; padding:16px; box-shadow:0 20px 35px rgba(0,0,0,0.5); }
    canvas { display:block; margin:0 auto; border-radius:28px; cursor:crosshair; box-shadow:0 8px 18px rgba(0,0,0,0.4); touch-action:none; }
    .info-panel { position:fixed; bottom:16px; right:20px; background:rgba(0,0,0,0.65); backdrop-filter:blur(4px); border-radius:40px; padding:8px 20px; color:#ffefb0; font-size:14px; font-weight:bold; font-family:monospace; pointer-events:none; z-index:10; border-left:4px solid #f5b642; }
  </style>
</head>
<body>
<div class="game-wrapper"><canvas id="gameCanvas" width="900" height="600"></canvas></div>
<div class="info-panel">🚩 旗帜(加速) | 🏃 撑杆跳 | 🛒 小推车 | 像素重制版</div>
<script>
(function() {
  "use strict";

  // ===================== 配置 =====================
  const CONFIG = {
    GRID: { rows:5, cols:9, cellW:80, cellH:80, startX:100, startY:120 },
    PLANTS: {
      pea:       { name:'豌豆射手', cost:100, cdMax:190, icon:'🌱', type:'pea', double:false, locked:false },
      doublepea: { name:'双发豌豆', cost:200, cdMax:380, icon:'🔥', type:'doublepea', double:true, locked:true },
      sun:       { name:'向日葵',   cost:50,  cdMax:180, icon:'🌻', type:'sun', double:false, locked:false },
      wall:      { name:'坚果墙',   cost:75,  cdMax:280, icon:'🌰', type:'wall', double:false, locked:false },
      cherry:    { name:'樱桃炸弹', cost:150, cdMax:1080,icon:'🍒', type:'cherry', double:false, locked:false },
      potato:    { name:'土豆地雷', cost:50,  cdMax:300, icon:'🥔', type:'potato', double:false, locked:false },
      chomper:   { name:'食人花',   cost:150, cdMax:400, icon:'🌱', type:'chomper', double:false, locked:false },
      icepea:    { name:'寒冰豌豆', cost:150, cdMax:220, icon:'❄️', type:'icepea', double:false, locked:false }
    },
    LEVELS: [
      { name:'白天草地', bgGradient:['#bbda7a','#8ab84a'], naturalSun:true, sunInterval:260, zombieHealthMult:1, zombieSpeedMult:1, totalWaves:2, allowBucket:false, allowPole:true, flagBuff:1.2, startSun:150, reward:'doublepea' },
      { name:'暗夜森林', bgGradient:['#7a9ab8','#4a6a8a'], naturalSun:false, sunInterval:520, zombieHealthMult:1, zombieSpeedMult:1, totalWaves:3, allowBucket:true, allowPole:true, flagBuff:1.2, startSun:150, reward:'sunBonus' },
      { name:'屋顶之夜', bgGradient:['#8a6ab8','#5a3a7a'], naturalSun:false, sunInterval:260, zombieHealthMult:1.3, zombieSpeedMult:1.1, totalWaves:4, allowBucket:true, allowPole:true, flagBuff:1.3, startSun:150, reward:null }
    ],
    ZOMBIE_HP: { normal:140, cone:240, bucket:400, flag:180, pole:160 },
    SUN_VALUE:25, SUNFLOWER_INTERVAL:880, POTATO_ARM_TIME:600, CHOMPER_COOLDOWN:600,
  };

  // ===================== 核心层 =====================
  class EventBus { constructor(){ this.events={}; } on(e,f){ (this.events[e]??=[]).push(f); } off(e,f){ if(this.events[e]) this.events[e]=this.events[e].filter(cb=>cb!==f); } emit(e,d){ this.events[e]?.forEach(cb=>cb(d)); } }
  const GameState = { MENU:'menu', LEVEL_SELECT:'levelSelect', PLAYING:'playing', PAUSED:'paused', GAME_OVER:'gameover', VICTORY:'victory' };
  class StateMachine {
    constructor(eb){ this.eb=eb; this.current=GameState.MENU; }
    transitionTo(s) {
      const allowed = {
        [GameState.MENU]:[GameState.LEVEL_SELECT],
        [GameState.LEVEL_SELECT]:[GameState.PLAYING,GameState.MENU],
        [GameState.PLAYING]:[GameState.PAUSED,GameState.GAME_OVER,GameState.VICTORY],
        [GameState.PAUSED]:[GameState.PLAYING],
        [GameState.GAME_OVER]:[GameState.MENU],
        [GameState.VICTORY]:[GameState.LEVEL_SELECT,GameState.MENU]
      }[this.current];
      if(allowed?.includes(s)){ const o=this.current; this.current=s; this.eb.emit('state:changed',{from:o,to:s}); return true; }
      return false;
    }
  }

  // ===================== 模型层 =====================
  class GameModel {
    constructor(eb,sm){ this.eb=eb; this.sm=sm; this.permanentUnlocks={doublepea:false,startSunBonus:0}; this.unlockedLevels=[true,false,false]; this.reset(); }
    reset(){
      this.sun=150; this.plants=[]; this.zombies=[]; this.bullets=[]; this.sunDrops=[]; this.explosions=[]; this.lawnmowers=[];
      this.cooldowns={}; this.currentWave=1; this.totalWaves=4; this.waveInProgress=false; this.waveZombiesToSpawn=0;
      this.waveSpawnTimer=0; this.waveZombiesRemaining=0; this.waveCompleted=false; this.flagZombieAlive=false;
      this.waveFlagSpawned=false; this.invasionProgress=0; this.naturalSunCounter=0; this.levelConfig=null;
      this.currentLevel=1; this.gameStartFrame=0; this.wavePhase=0; this.phaseTimer=0;
    }
    loadLevel(idx){
      this.reset(); this.currentLevel=idx+1; const cfg=CONFIG.LEVELS[idx]; this.levelConfig={...cfg};
      this.totalWaves=cfg.totalWaves; this.sun=cfg.startSun+(this.permanentUnlocks.startSunBonus||0);
      CONFIG.PLANTS.doublepea.locked=!this.permanentUnlocks.doublepea;
      for(let r=0;r<CONFIG.GRID.rows;r++) this.lawnmowers.push({row:r,active:false,used:false,x:30});
      Object.keys(CONFIG.PLANTS).forEach(k=>this.cooldowns[k]=0);
      this.eb.emit('level:loaded',cfg);
    }
    getWaveCount(){ let b=Math.max(3,3+this.currentWave); if(this.currentLevel===3) b=Math.floor(b*1.5); return b; }
    startWave(){
      if(this.currentWave>this.totalWaves) return; this.waveInProgress=true;
      const n=this.getWaveCount(); this.waveZombiesToSpawn=n; this.waveZombiesRemaining=n+1;
      this.waveSpawnTimer=0; this.waveFlagSpawned=false; this.flagZombieAlive=false;
    }
    spawnWaveZombie(){
      if(this.waveZombiesToSpawn<=0) return;
      let rc=new Array(CONFIG.GRID.rows).fill(0); this.zombies.forEach(z=>{if(z.row>=0)rc[z.row]++;});
      let mr=0; for(let i=1;i<CONFIG.GRID.rows;i++) if(rc[i]<rc[mr]) mr=i;
      let flag=!this.waveFlagSpawned && !this.zombies.some(z=>z.type==='flag');
      if(flag) this.waveFlagSpawned=true;
      let z;
      if(flag) z=new FlagZombie(mr,true,this.levelConfig);
      else if(this.levelConfig.allowPole && Math.random()<0.3 && this.wavePhase>=2) z=new PoleVaultZombie(mr,true,this.levelConfig);
      else z=new Zombie(mr,true,this.levelConfig);
      this.zombies.push(z); this.waveZombiesToSpawn--;
    }
    finishWave(){
      if(!this.waveInProgress) return; this.waveInProgress=false; this.waveZombiesToSpawn=0;
      if(this.waveZombiesRemaining<=0 && this.currentWave<this.totalWaves && !this.waveCompleted){
        this.waveCompleted=true; this.currentWave++; this.eb.emit('wave:finished',{nextWave:this.currentWave});
      }
    }
    checkVictory(){ return this.currentWave===this.totalWaves && !this.waveInProgress && this.zombies.length===0; }
    applyReward(){
      const r=this.levelConfig.reward; if(r==='doublepea') this.permanentUnlocks.doublepea=true; else if(r==='sunBonus') this.permanentUnlocks.startSunBonus=50;
      if(this.currentLevel<CONFIG.LEVELS.length) this.unlockedLevels[this.currentLevel]=true;
    }
    update(gf){
      if(!this.levelConfig) return;
      Object.keys(this.cooldowns).forEach(k=>{if(this.cooldowns[k]>0)this.cooldowns[k]--;});
      if(this.levelConfig.naturalSun){
        if(++this.naturalSunCounter>=this.levelConfig.sunInterval){ this.naturalSunCounter=0; this.sunDrops.push(new SunDrop(Math.floor(Math.random()*5), Math.floor(Math.random()*9))); }
      }
      this.plants.forEach(p=>p.update(this));
      for(let i=this.bullets.length-1;i>=0;i--){ this.bullets[i].update(); if(this.bullets[i].x>950) this.bullets.splice(i,1); }
      for(let i=this.zombies.length-1;i>=0;i--){
        const z=this.zombies[i], before=z.hp; z.update(this);
        if(z.hp<=0 && before>0) this.eb.emit('zombie:killed',{zombie:z});
        if(z.hp<=0) this.zombies.splice(i,1);
      }
      for(let i=this.bullets.length-1;i>=0;i--){
        const b=this.bullets[i]; let hit=false;
        for(const z of this.zombies){ if(z.row===b.row&&b.x+8>z.x&&b.x-8<z.x+50){ z.hp-=b.damage; if(b instanceof IceBullet) z.applySlow(); hit=true; break; } }
        if(hit) this.bullets.splice(i,1);
      }
      for(let i=this.sunDrops.length-1;i>=0;i--) if(!this.sunDrops[i].update()) this.sunDrops.splice(i,1);
      this.plants=this.plants.filter(p=>p.hp>0);
      for(let i=this.explosions.length-1;i>=0;i--) if(--this.explosions[i].timer<=0) this.explosions.splice(i,1);
      this.lawnmowers.forEach(m=>{
        if(m.used) return;
        this.zombies.forEach(z=>{ if(z.row===m.row && z.hp>0 && z.x<=m.x+40){ m.active=true; m.used=true; this.zombies.forEach(z2=>{if(z2.row===m.row)z2.hp=0;}); this.eb.emit('lawnmower:triggered',{row:m.row}); } });
        if(m.active){ m.animFrame++; m.x+=12; if(m.x>950) m.active=false; }
      });
      const flag=this.zombies.find(z=>z.type==='flag'&&z.hp>0);
      if(flag){ if(!this.flagZombieAlive){ this.flagZombieAlive=true; this.zombies.forEach(z=>{if(z.type!=='flag'&&z.hp>0){ z.setSpeedBoost(this.levelConfig.flagBuff); z.speedBoosted=true; }}); } }
      else if(this.flagZombieAlive){ this.flagZombieAlive=false; this.zombies.forEach(z=>{if(z.type!=='flag'&&z.speedBoosted){ z.resetSpeed(this.levelConfig.flagBuff); z.speedBoosted=false; }}); }
      let p=(this.currentWave-1)/this.totalWaves;
      if(this.waveInProgress && this.waveZombiesToSpawn>=0) p+=0.5/this.totalWaves;
      this.invasionProgress=Math.min(0.99,p);
      if(this.currentWave===this.totalWaves && !this.waveInProgress && this.zombies.length===0) this.invasionProgress=1;
    }
  }

  // 实体类 (同前，省略)
  class SunDrop { constructor(r,c){ this.row=r; this.col=c; this.x=CONFIG.GRID.startX+c*CONFIG.GRID.cellW+CONFIG.GRID.cellW/2; this.y=CONFIG.GRID.startY+r*CONFIG.GRID.cellH+CONFIG.GRID.cellH/2; this.lifeTimer=600; } update(){ this.lifeTimer--; return this.lifeTimer>0; } }
  class Plant {
    constructor(r,c,t,isD=false){ this.row=r; this.col=c; this.type=t; this.isDouble=isD; this.x=CONFIG.GRID.startX+c*CONFIG.GRID.cellW; this.y=CONFIG.GRID.startY+r*CONFIG.GRID.cellH; this.hp=t==='wall'?380:(t==='sun'?90:(t==='cherry'?60:110)); this.maxHp=this.hp; this.shootTimer=0; this.sunGenTimer=0; this.explodeTriggered=false; this.armed=false; this.armTimer=0; this.eatTimer=0; }
    update(m){
      if(this.type==='potato'){ if(!this.armed){ if(++this.armTimer>=600) this.armed=true; } if(this.armed){ for(const z of m.zombies){ if(z.row===this.row && z.x+30>this.x && z.x<this.x+70){ m.explosions.push({x:this.x+35,y:this.y+35,timer:16}); m.zombies.forEach(z2=>{if(z2.row===this.row&&z2.x+40>this.x&&z2.x<this.x+80)z2.hp=0;}); const i=m.plants.indexOf(this); if(i!==-1)m.plants.splice(i,1); break; } } } }
      else if(this.type==='chomper'){ if(this.eatTimer>0){ this.eatTimer--; } else { for(const z of m.zombies){ if(z.row===this.row && z.x+35>this.x && z.x<this.x+65){ z.hp=0; this.eatTimer=CONFIG.CHOMPER_COOLDOWN; m.explosions.push({x:this.x+35,y:this.y+35,timer:8}); break; } } } }
      else if((this.type==='pea'||this.type==='doublepea') && m.zombies.some(z=>z.row===this.row&&z.hp>0)){ if(++this.shootTimer>=220){ this.shootTimer=0; if(this.isDouble){ m.bullets.push(new Bullet(this.x+58,this.y+30,this.row)); setTimeout(()=>{if(m.plants.includes(this))m.bullets.push(new Bullet(this.x+58,this.y+42,this.row));},80); } else m.bullets.push(new Bullet(this.x+58,this.y+35,this.row)); } }
      else if(this.type==='icepea' && m.zombies.some(z=>z.row===this.row&&z.hp>0)){ if(++this.shootTimer>=240){ this.shootTimer=0; m.bullets.push(new IceBullet(this.x+58,this.y+35,this.row)); } }
      else if(this.type==='sun'){ if(++this.sunGenTimer>=CONFIG.SUNFLOWER_INTERVAL){ this.sunGenTimer=0; m.sunDrops.push(new SunDrop(this.row,this.col)); } }
      else if(this.type==='cherry' && !this.explodeTriggered){ this.explodeTriggered=true; setTimeout(()=>{ const i=m.plants.indexOf(this); if(i!==-1){ m.explosions.push({x:this.x+35,y:this.y+35,timer:16}); m.zombies.forEach(z=>{ const zc=Math.floor((z.x-CONFIG.GRID.startX)/CONFIG.GRID.cellW); if(Math.abs(z.row-this.row)<=1&&Math.abs(zc-this.col)<=1) z.hp=0; }); m.plants.splice(i,1); } },100); }
    }
  }
  class Bullet { constructor(x,y,r,d=32){ this.x=x;this.y=y;this.row=r;this.damage=d; } update(){ this.x+=5.2; } }
  class IceBullet extends Bullet { constructor(x,y,r){ super(x,y,r,25); } }
  class Zombie {
    constructor(r,isW,cfg){ this.row=r; this.x=920; this.y=CONFIG.GRID.startY+r*CONFIG.GRID.cellH+8; let types=['normal','cone']; if(cfg.allowBucket) types.push('bucket'); let t=types[Math.floor(Math.random()*types.length)]; this.type=t; this.hp=Math.floor((CONFIG.ZOMBIE_HP[t]||140)*cfg.zombieHealthMult); this.maxHp=this.hp; this.baseSpeed=(0.11+Math.random()*0.05)*cfg.zombieSpeedMult; this.speed=this.baseSpeed; this.attackTimer=0; this.attackDamage=22; this.isWaveMember=isW; this.slowTimer=0; this.speedBoosted=false; this.lcfg=cfg; }
    update(m){
      if(this.slowTimer>0){ this.slowTimer--; this.speed=this.baseSpeed*0.5; } else this.speed=this.baseSpeed;
      const p=m.plants.find(p=>p.row===this.row&&this.x+20>p.x&&this.x<p.x+65);
      if(p){ if(++this.attackTimer>26){ p.hp-=this.attackDamage; this.attackTimer=0; } } else this.x-=this.speed;
      if(this.x<40) m.sm.transitionTo(GameState.GAME_OVER);
    }
    applySlow(){ this.slowTimer=180; }
    setSpeedBoost(b){ this.baseSpeed*=b; this.speed=this.baseSpeed; }
    resetSpeed(b){ this.baseSpeed/=b; this.speed=this.baseSpeed; }
  }
  class FlagZombie extends Zombie { constructor(r,i,cfg){ super(r,i,cfg); this.type='flag'; this.hp=Math.floor(CONFIG.ZOMBIE_HP.flag*cfg.zombieHealthMult); this.maxHp=this.hp; this.baseSpeed=0.14*cfg.zombieSpeedMult; this.speed=this.baseSpeed; } }
  class PoleVaultZombie extends Zombie {
    constructor(r,i,cfg){ super(r,i,cfg); this.type='pole'; this.hp=Math.floor(CONFIG.ZOMBIE_HP.pole*cfg.zombieHealthMult); this.maxHp=this.hp; this.baseSpeed=0.17*cfg.zombieSpeedMult; this.speed=this.baseSpeed; this.hasJumped=false; }
    update(m){
      if(this.slowTimer>0){ this.slowTimer--; this.speed=this.baseSpeed*0.5; } else this.speed=this.baseSpeed;
      if(!this.hasJumped){ const p=m.plants.find(p=>p.row===this.row&&this.x+35>p.x&&this.x<p.x+65); if(p){ this.hasJumped=true; this.baseSpeed=0.11*this.lcfg.zombieSpeedMult; this.speed=this.baseSpeed; m.explosions.push({x:this.x+26,y:this.y+20,timer:6}); this.x=p.x+70; return; } }
      super.update(m);
    }
  }

  // ===================== 像素 UI 视图 =====================
  const ctx=document.getElementById('gameCanvas').getContext('2d');
  // 像素字体设置
  const PIXEL_FONT = 'bold 10px "Press Start 2P", monospace';

  // 像素辅助函数
  function pixelRect(x,y,w,h,color) { ctx.fillStyle=color; ctx.fillRect(x,y,w,h); }
  function pixelBorder(x,y,w,h,color,thickness=2) { ctx.strokeStyle=color; ctx.lineWidth=thickness; ctx.strokeRect(x,y,w,h); }
  function pixelText(text,x,y,color,size=10) { ctx.font=`bold ${size}px "Press Start 2P", monospace`; ctx.fillStyle=color; ctx.fillText(text,x,y); }

  // 绘制像素太阳
  function drawPixelSun(cx,cy,r,alpha=1) {
    ctx.globalAlpha=alpha;
    const colors=['#FFD700','#FFA500','#FF8C00'];
    for(let i=0;i<3;i++) { pixelRect(cx-r+i*2,cy-r-2,4,4,colors[i]); pixelRect(cx+r-i*2,cy-r-2,4,4,colors[i]); }
    pixelRect(cx-2,cy-r-4,4,8,colors[0]);
    pixelRect(cx-r,cy-2,2*r,4,colors[0]);
    ctx.globalAlpha=1;
  }

  // 像素铲子
  function drawPixelShovel(x,y) {
    ctx.fillStyle='#8B5A2B'; ctx.fillRect(x+10,y,4,16); // 柄
    ctx.fillStyle='#A0522D'; ctx.fillRect(x+8,y+12,8,6); ctx.fillRect(x+6,y+18,12,4);
    ctx.fillStyle='#CD853F'; ctx.fillRect(x+10,y+14,4,4); // 高光
    ctx.fillStyle='#696969'; ctx.fillRect(x+4,y+22,16,3); // 金属头
    ctx.fillStyle='#A9A9A9'; ctx.fillRect(x+6,y+25,12,2);
  }

  const View = {
    drawMenu(){
      ctx.fillStyle='#1a2a1a'; ctx.fillRect(0,0,900,600);
      // 像素砖块背景
      for(let y=0;y<600;y+=32) for(let x=0;x<900;x+=32) { pixelRect(x,y,32,32,'#2a3a2a'); pixelRect(x+2,y+2,28,28,'#1e2e1e'); }
      pixelText('🌱 植物大战僵尸',190,200,'#FFD700',28);
      pixelText('像素重制版 · 冒险模式',250,260,'#ADFF2F',16);
      // 像素按钮
      const bx=320,by=340,bw=260,bh=70;
      for(let i=0;i<bh;i+=8) { pixelRect(bx,by+i,bw,8,i%16===0?'#8B4513':'#A0522D'); }
      pixelBorder(bx,by,bw,bh,'#FFD700',4);
      pixelText('开始冒险',bx+50,by+50,'#FFE4B5',18);
      pixelText('按 ESC 暂停  |  R 重置',250,500,'#FFEFB0',10);
    },
    drawLevelSelect(mo){
      ctx.fillStyle='#1a2a1a'; ctx.fillRect(0,0,900,600);
      pixelText('选择关卡',300,100,'#FFD700',32);
      for(let i=0;i<CONFIG.LEVELS.length;i++){
        let x=150+i*220, y=200;
        let unlocked=mo.unlockedLevels[i];
        // 卡片背景
        for(let j=0;j<220;j+=16) { pixelRect(x,y+j,160,16,j%32===0?'#4B3E2A':'#3A2E1A'); }
        pixelBorder(x,y,160,220,unlocked?'#FFD700':'#696969',4);
        pixelText(`${i+1}`,x+60,y+80,unlocked?'#FFD700':'#808080',24);
        pixelText(CONFIG.LEVELS[i].name,x+20,y+130,unlocked?'#FFE4B5':'#888',10);
        if(!unlocked){ pixelText('🔒',x+50,y+180,'#B22222',28); }
      }
      // 返回按钮
      const rbx=350,rby=480,rbw=200,rbh=60;
      for(let i=0;i<rbh;i+=8) pixelRect(rbx,rby+i,rbw,8,i%16===0?'#8B4513':'#A0522D');
      pixelBorder(rbx,rby,rbw,rbh,'#FFD700',4);
      pixelText('返回',rbx+60,rby+40,'#FFE4B5',18);
    },
    drawGame(m){
      let g=ctx.createLinearGradient(0,100,0,500);
      g.addColorStop(0,m.levelConfig.bgGradient[0]); g.addColorStop(1,m.levelConfig.bgGradient[1]);
      ctx.fillStyle=g; ctx.fillRect(0,0,900,600);
      // 草地格
      for(let r=0;r<5;r++) for(let c=0;c<9;c++){ pixelRect(100+c*80,120+r*80,80,80,'#7CFC00'); pixelRect(102+c*80,122+r*80,76,76,'#3CB371'); }
      m.plants.forEach(p=>this.drawPlant(p));
      m.zombies.forEach(z=>this.drawZombie(z));
      m.bullets.forEach(b=>{ pixelRect(b.x-3,b.y-3,6,6,'#FFD700'); });
      m.sunDrops.forEach(s=>{ drawPixelSun(s.x,s.y,12,s.lifeTimer/600); });
      m.lawnmowers.forEach(l=>{ if(!l.used||l.active){ let y=120+l.row*80+40-15; pixelRect(l.x,y,50,30,'#8B4513'); pixelRect(l.x+5,y-5,40,10,'#A0522D'); pixelRect(l.x+10,y+30,8,8,'#333'); pixelRect(l.x+40,y+30,8,8,'#333'); } });
      m.explosions.forEach(e=>{ ctx.fillStyle=`rgba(255,69,0,${e.timer/16})`; ctx.beginPath(); ctx.arc(e.x,e.y,32,0,Math.PI*2); ctx.fill(); });
      this.drawUI(m);
      this.drawProgressBar(m);
    },
    drawPlant(p){
      // 像素植物（简化，用色块表示）
      const x=p.x+35, y=p.y+40;
      pixelRect(x-16,y-16,32,32,'#228B22');
      if(p.type==='pea') { pixelRect(x+12,y-8,16,16,'#ADFF2F'); }
      else if(p.type==='doublepea') { pixelRect(x+12,y-12,16,16,'#FF8C00'); pixelRect(x+12,y+4,16,8,'#FF8C00'); }
      else if(p.type==='sun') { drawPixelSun(x,y,14,1); }
      else if(p.type==='wall') { pixelRect(x-12,y-12,24,24,'#D2B48C'); pixelText(Math.floor(p.hp),x-6,y+4,'black',8); }
      else if(p.type==='cherry') { pixelRect(x-12,y-12,12,24,'#DC143C'); pixelRect(x,y-12,12,24,'#DC143C'); }
      else if(p.type==='potato') { pixelRect(x-12,y-6,24,12,'#8B4513'); if(p.armed) pixelText('💣',x-8,y+10,'white',12); }
      else if(p.type==='chomper') { pixelRect(x-10,y-10,20,20,'#556B2F'); if(p.eatTimer>0) pixelRect(x-12,y-12,24,24,'rgba(0,0,0,0.5)'); }
      else if(p.type==='icepea') { pixelRect(x+12,y-8,16,16,'#00BFFF'); }
    },
    drawZombie(z){
      const x=z.x+26, y=120+z.row*80+40;
      pixelRect(x-14,y-14,28,28,'#6B8E23');
      pixelText(z.type==='flag'?'🚩':z.type==='pole'?'🏃':z.type==='cone'?'⛑️':z.type==='bucket'?'🪣':'🧟',x-8,y+8,'white',12);
      pixelRect(x-14,y-22,28,4,'#B22222');
      pixelRect(x-14,y-22,28*z.hp/z.maxHp,4,'#00FF00');
    },
    drawUI(m){
      // 顶部工具栏
      ctx.fillStyle='#2C1A0A'; ctx.fillRect(0,0,900,105);
      pixelBorder(0,0,900,105,'#FFD700',3);
      // 阳光计数器
      drawPixelSun(780,45,20,1);
      pixelText(`${m.sun}`,810,52,'#FFD700',18);
      // 卡片
      const keys=Object.keys(CONFIG.PLANTS);
      for(let i=0;i<keys.length;i++){
        let k=keys[i], card=CONFIG.PLANTS[k], x=8+i*68, w=62, h=88;
        // 像素卡片底板
        for(let j=0;j<h;j+=8) pixelRect(x,8+j,w,8,j%16===0?'#8B7355':'#6B5A4A');
        pixelBorder(x,8,w,h,'#FFD700',2);
        // 植物图标（小像素块）
        let iconX=x+10, iconY=25;
        if(k==='pea') { pixelRect(iconX+20,iconY,12,12,'#ADFF2F'); }
        else if(k==='doublepea') { pixelRect(iconX+20,iconY-4,12,10,'#FF8C00'); pixelRect(iconX+20,iconY+8,12,8,'#FF8C00'); }
        else if(k==='sun') { drawPixelSun(iconX+26,iconY+6,8,1); }
        else if(k==='wall') { pixelRect(iconX+18,iconY,16,16,'#D2B48C'); }
        else if(k==='cherry') { pixelRect(iconX+18,iconY,8,16,'#DC143C'); pixelRect(iconX+26,iconY,8,16,'#DC143C'); }
        else if(k==='potato') { pixelRect(iconX+18,iconY+6,16,10,'#8B4513'); }
        else if(k==='chomper') { pixelRect(iconX+20,iconY,12,12,'#556B2F'); }
        else if(k==='icepea') { pixelRect(iconX+20,iconY,12,12,'#00BFFF'); }

        pixelText(card.name,x+5,75,'#FFE4B5',6);
        pixelText(`$${card.cost}`,x+40,85,'#FFD700',8);
        // 冷却遮罩
        if(!card.locked && m.cooldowns[k]>0){
          let cd=m.cooldowns[k]/card.cdMax;
          ctx.fillStyle='rgba(0,0,0,0.7)'; ctx.fillRect(x,8+h*(1-cd),w,h*cd);
        }
        if(card.locked){
          ctx.fillStyle='rgba(0,0,0,0.7)'; ctx.fillRect(x,8,w,h);
          pixelText('🔒',x+22,60,'#B22222',18);
        }
      }
      // 铲子
      let sx=560,sy=8;
      for(let j=0;j<88;j+=8) pixelRect(sx,sy+j,70,8,j%16===0?'#8B7355':'#6B5A4A');
      pixelBorder(sx,sy,70,88,'#FFD700',2);
      drawPixelShovel(sx+25,sy+20);
    },
    drawProgressBar(m){
      let bx=580,by=540,bw=280,bh=26;
      pixelRect(bx,by,bw,bh,'#1A1A1A');
      for(let i=0;i<bh;i+=4) pixelRect(bx,bx+i,bw*m.invasionProgress,4,'#FF4500');
      pixelBorder(bx,by,bw,bh,'#FFD700',2);
      for(let w=1; w<=m.totalWaves; w++){
        let nx=bx+(w/m.totalWaves)*bw;
        pixelRect(nx-2,by-4,4,30,'#FFD700');
        pixelText(`${w}`,nx-6,by-6,'#FFD700',8);
      }
      pixelText('入侵进度',bx+90,by-8,'#FFE4B5',10);
    },
    drawOverlay(txt,sub){
      ctx.fillStyle='rgba(0,0,0,0.85)'; ctx.fillRect(0,0,900,600);
      pixelText(txt,260,280,'#FF4500',32);
      if(sub) pixelText(sub,290,380,'#FFE4B5',16);
    }
  };

  // 控制器 (InputManager) 保持不变，但因篇幅省略，直接使用前面逻辑
  // 这里重新写入完整 InputManager
  class InputManager {
    constructor(cv,m,sm,eb){ this.cv=cv; this.m=m; this.sm=sm; this.eb=eb; this.sel='pea'; this.shovel=false; this.cv.addEventListener('click',e=>this.clk(e)); this.cv.addEventListener('touchstart',e=>{e.preventDefault();this.clk(e);}); window.addEventListener('keydown',e=>this.key(e)); }
    grid(px,py){ let c=Math.floor((px-100)/80), r=Math.floor((py-120)/80); return (r>=0&&r<5&&c>=0&&c<9)?{row:r,col:c}:null; }
    clk(e){
      const r=this.cv.getBoundingClientRect(), cx=(e.clientX-r.left)*(900/r.width), cy=(e.clientY-r.top)*(600/r.height);
      const s=this.sm.current;
      if(s===GameState.MENU){ if(cx>=320&&cx<=580&&cy>=340&&cy<=410) this.sm.transitionTo(GameState.LEVEL_SELECT); }
      else if(s===GameState.LEVEL_SELECT){
        if(cx>=350&&cx<=550&&cy>=480&&cy<=540) this.sm.transitionTo(GameState.MENU);
        for(let i=0;i<3;i++){ let x=150+i*220; if(cx>=x&&cx<=x+160&&cy>=200&&cy<=420&&this.m.unlockedLevels[i]){ this.m.loadLevel(i); this.sm.transitionTo(GameState.PLAYING); } }
      }
      else if(s===GameState.PLAYING){
        const g=this.grid(cx,cy); if(g) this.collectSun(g);
        if(this.plantSel(cx,cy)) return;
        if(g) this.place(g);
      }
      else if(s===GameState.GAME_OVER||s===GameState.VICTORY){ this.sm.transitionTo(GameState.MENU); }
      else if(s===GameState.PAUSED){ this.sm.transitionTo(GameState.PLAYING); }
    }
    collectSun(g){ for(let i=this.m.sunDrops.length-1;i>=0;i--){ let s=this.m.sunDrops[i]; if(s.row===g.row&&s.col===g.col){ this.m.sun+=25; this.m.sunDrops.splice(i,1); break; } } }
    plantSel(cx,cy){
      if(cy>=8&&cy<=96){
        if(cx>=560&&cx<=630){ this.shovel=!this.shovel; if(this.shovel) this.sel=null; else if(!this.sel) this.sel='pea'; return true; }
        if(cx>=8&&cx<=8+8*68){ let idx=Math.floor((cx-8)/68), keys=Object.keys(CONFIG.PLANTS); if(idx>=0&&idx<keys.length){ let k=keys[idx]; if(!CONFIG.PLANTS[k].locked){ this.shovel=false; this.sel=k; return true; } } }
      }
      return false;
    }
    place(g){
      if(this.shovel){ let i=this.m.plants.findIndex(p=>p.row===g.row&&p.col===g.col); if(i!==-1) this.m.plants.splice(i,1); }
      else if(this.sel){
        if(this.m.plants.some(p=>p.row===g.row&&p.col===g.col)) return;
        let cd=CONFIG.PLANTS[this.sel]; if(cd&&!cd.locked && this.m.cooldowns[this.sel]===0 && this.m.sun>=cd.cost){
          this.m.plants.push(new Plant(g.row,g.col,this.sel,cd.double));
          this.m.sun-=cd.cost; this.m.cooldowns[this.sel]=cd.cdMax; this.sel=null;
        }
      }
    }
    key(e){
      if(e.key==='Escape'){ let s=this.sm.current; if(s===GameState.PLAYING) this.sm.transitionTo(GameState.PAUSED); else if(s===GameState.PAUSED) this.sm.transitionTo(GameState.PLAYING); }
      if((e.key==='r'||e.key==='R') && this.sm.current===GameState.PLAYING) this.m.loadLevel(this.m.currentLevel-1);
    }
  }

  // 关卡管理
  class LevelManager {
    constructor(m,eb){ this.m=m; this.eb=eb; this.eb.on('wave:finished',()=>{ setTimeout(()=>{ if(this.m.wavePhase>=3) this.m.startWave(); },200); }); }
    update(gf){
      if(this.m.wavePhase>=3){
        if(this.m.waveInProgress && this.m.waveZombiesToSpawn>0){ if(++this.m.waveSpawnTimer>=32){ this.m.waveSpawnTimer=0; this.m.spawnWaveZombie(); } }
        else if(this.m.waveInProgress){ let a=this.m.zombies.filter(z=>z.isWaveMember).length; this.m.waveZombiesRemaining=a; if(a===0 && this.m.waveZombiesToSpawn===0) this.m.finishWave(); }
        return;
      }
      let el=gf-this.m.gameStartFrame;
      if(this.m.wavePhase===0 && el>=720){ this.m.wavePhase=1; this.m.phaseTimer=0; }
      else if(this.m.wavePhase===1){ if(this.m.phaseTimer<540){ if(this.m.phaseTimer%340===30 && this.m.phaseTimer>0) this.spawnRoam(); this.m.phaseTimer++; } else { this.m.wavePhase=2; this.m.phaseTimer=0; } }
      else if(this.m.wavePhase===2){ if(this.m.phaseTimer<650){ if(this.m.phaseTimer%190===40 && this.m.phaseTimer>0) this.spawnRoam(); this.m.phaseTimer++; } else { this.m.wavePhase=3; this.m.startWave(); } }
    }
    spawnRoam(){ if(this.m.zombies.filter(z=>!z.isWaveMember).length>=5) return; let r=Math.floor(Math.random()*5), z=this.m.levelConfig.allowPole && Math.random()<0.3 ? new PoleVaultZombie(r,false,this.m.levelConfig) : new Zombie(r,false,this.m.levelConfig); this.m.zombies.push(z); }
  }

  // 主循环
  const eb=new EventBus(), sm=new StateMachine(eb), model=new GameModel(eb,sm);
  new InputManager(document.getElementById('gameCanvas'),model,sm,eb);
  const lm=new LevelManager(model,eb);
  let gf=0;
  function loop(){
    if(sm.current===GameState.PLAYING){
      gf++; if(model.gameStartFrame===0) model.gameStartFrame=gf;
      lm.update(gf); model.update(gf);
      if(model.checkVictory()){ model.applyReward(); sm.transitionTo(GameState.VICTORY); }
    }
    if(sm.current===GameState.MENU) View.drawMenu();
    else if(sm.current===GameState.LEVEL_SELECT) View.drawLevelSelect(model);
    else { View.drawGame(model); if(sm.current===GameState.GAME_OVER) View.drawOverlay('💀 GAME OVER 💀','点击返回菜单'); else if(sm.current===GameState.VICTORY) View.drawOverlay('🏆 胜利！ 🏆','点击返回菜单'); else if(sm.current===GameState.PAUSED) View.drawOverlay('⏸ 暂停','按 ESC 继续'); }
    requestAnimationFrame(loop);
  }
  loop();
})();
</script>
</body>
</html>
