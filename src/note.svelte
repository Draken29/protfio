<!-- <script>
  import './app.css';
  import { gsap } from "gsap";
  import { onMount } from "svelte";


  onMount(() => {

   gsap.set(".lens", {xPercent: -50, yPercent: -50});

   let xTo = gsap.quickTo(".lens", "x", {duration: 0.3, ease: "power3"}),
    yTo = gsap.quickTo(".lens", "y", {duration: 0.3, ease: "power3"});

window.addEventListener("mousemove", e => {
  xTo(e.clientX);
  yTo(e.clientY);
});

  });
</script>

<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Alexandria:wght@100..900&family=Momo+Signature&family=Noto+Sans+Syriac+Western:wght@100..900&display=swap" rel="stylesheet">

<main class="h-screen">
  <div class=" w-screen flex justify-center algin-center overflow-hidden ">
  <h1 class=" momo-signature-regular text-[5rem] text-white  ">
    <div class="absolute top-[30%] left-[30%]">
    JoveLine 
  </div>
  <div class="absolute top-[40%] left-[40%]">
    Programming
  </div>
  </h1>
  <div class="lens"></div>
</div>

  <div class="flex justify-center items-center">
  <div class=" w-screen  relative blur-3xl">
    <div class="blob1 absolute w-[700px] h-[400px] rounded-full opacity-90 bg-linear-to-br from-[#43a4a2] to-[#0a0063] "></div> 
  </div>

  <div class='w-screen  relative blur-3xl '>
    <div class="blob2 absolute w-[500px] h-[500px] rounded-full opacity-90 bg-linear-to-br from-[#70579e94] to-[#5500ff] "></div> 
  </div>
  </div>
   
  
    
</main>

<style>


.momo-signature-regular {
  font-family: "Momo Signature", cursive;
  font-weight: 400;
  font-style: normal;
}


h1 {
  z-index: 1;
}

.lens {
  width: 150px;
  height: 150px;
  position: fixed;
  top: 0;
  left: 0;
  border-radius: 50%;
  pointer-events: none;
  border: 2px solid rgba(255,255,255,0.7);
  backdrop-filter: blur(5px); 
  -webkit-backdrop-filter: blur(5px);
  z-index: 10;
}






.blob1{
  animation: move 10s infinite ease-in-out;
}

.blob2{
  animation: move2 8s infinite ease-in-out;
}




@keyframes move {
  0%   { transform: translate(-40%, -40%) scale(1); }
  33%  { transform: translate(-30%, -30%) scale(1.3); }
  66%  { transform: translate(20%, 40%) scale(0.8); }
  100% { transform: translate(-60%, -20%) scale(1); }
}


@keyframes move2{
  0%   { transform: translate(-20%, 30%) scale(1); }
  33%  { transform: translate(-20%, 30%) scale(1.3); }
  66%  { transform: translate(20%, -40%) scale(0.8); }
  100% { transform: translate(-70%, 30%) scale(1); }
}
</style> -->


<!-- <script>
import { onMount } from "svelte";
import { gsap } from "gsap";
import { Draggable, InertiaPlugin, Physics2DPlugin } from "gsap/all";

gsap.registerPlugin(Draggable, InertiaPlugin, Physics2DPlugin);

onMount(() => {
  console.clear();

  const $stage = document.querySelector('.stage');
  const stageSize = { w: $stage.clientWidth, h: $stage.clientHeight };

  const resizeObserver = new ResizeObserver((entries) => {
    const { width, height } = entries[0].contentRect;
    stageSize.w = Math.round(width);
    stageSize.h = Math.round(height);
  });
  resizeObserver.observe($stage);

  // utils
  const distance = (x1,y1,x2,y2) => Math.hypot(x2-x1,y2-y1);
  const length = (x,y) => Math.hypot(x,y);
  const angle = (x1,y1,x2,y2) => Math.atan2(y2-y1,x2-x1)*180/Math.PI;

  // particles
  const spawnParticle = ({ className, cssVars={}, text='', startX, startY, scale=1, duration, delay=0, velocity, angle, gravity }) => {
    const $el = document.createElement('div');
    $el.classList.add(className);
    $el.innerText = text;
    Object.keys(cssVars).forEach(k => $el.style[k]=cssVars[k]);

    gsap.set($el,{ x:startX, y:startY, xPercent:-50, yPercent:-50, scale });
    const tl = gsap.timeline({
      delay,
      onStart: ()=>$stage.appendChild($el),
      onComplete: ()=>$el.remove()
    });
    tl.to($el,{ duration, physics2D:{velocity, angle, gravity} },0);
    tl.to($el,{ duration, opacity:0 },0);
  }

  // Creature
  const CreatureStates = {
    spawning:'spawning',
    idle:'idle',
    pulling:'pulling',
    dragging:'dragging',
    dropping:'dropping',
    leaving:'leaving'
  };

  const createGroup = ({color='yellow', size=80, leg=40}) => {
    const html = `
      <div class="group" style="--color:${color}; --leg:${leg}px; --size:${size}px;">
        <div class="dragger"></div>
        <div class="creature">
          <div class="leg"></div>
          <div class="leg"></div>
          <div class="body"></div>
        </div>
      </div>`;
    const template = document.createElement('div');
    template.innerHTML = html;
    return template.querySelector('.group');
  }

  // States
  class CreatureState {
    constructor(creature){ this.creature = creature; }
    onEnter(fromState){}
    onExit(toState){}
  }

  class CreatureIdleState extends CreatureState {
    onEnter(fromState){
      if(fromState===CreatureStates.spawning) this.spawningToIdle();
      else if(fromState===CreatureStates.pulling) this.pullingToIdle();
    }
    spawningToIdle(){
      this.transition?.kill();
      const tl = gsap.timeline({ onComplete: this.playIdleAnimation });
      tl.fromTo(this.creature.$el,{scaleX:0, scaleY:0},{scaleX:1, scaleY:1, ease:'elastic.out', duration:gsap.utils.random(0.8,1)},0);
      this.transition=tl;
    }
    pullingToIdle(){
      this.transition?.kill();
      const tl = gsap.timeline({ onComplete: this.playIdleAnimation });
      tl.set(this.creature.$dragger,{x:this.creature.startX, y:this.creature.startY});
      tl.to(this.creature.$el,{scaleX:1, scaleY:1, ease:'elastic.out', duration:1},0);
      tl.set(this.creature.$el,{rotation:0});
      this.transition=tl;
    }
    playIdleAnimation = () => {
      const tl = gsap.timeline({ repeat:-1 });
      tl.add(()=>{ for(let i=0;i<3;i++) spawnParticle({
        className:'snooze-particle',
        text:'Z',
        startX:this.creature.startX+20,
        startY:this.creature.startY-20,
        velocity:gsap.utils.random(90,110),
        angle:gsap.utils.random(-55,-65),
        gravity:-100,
        duration:2,
        delay:i*0.25
      }); },0.5);
      tl.to(this.creature.$el,{scaleX:1.1, scaleY:0.9, duration:2},0.25);
      tl.to(this.creature.$el,{scaleX:1, scaleY:1, duration:1},2.5);
      this.idleAnimation = tl;
    }
    onExit(toState){ this.idleAnimation?.kill(); this.transition?.kill(); }
  }

  class CreaturePullingState extends CreatureState {
    onEnter(){ gsap.ticker.add(this.tick); }
    onExit(){ gsap.ticker.remove(this.tick); }
    tick = () => {
      const d = distance(this.creature.startX,this.creature.startY,this.creature.dragX,this.creature.dragY);
      const a = angle(this.creature.startX,this.creature.startY,this.creature.dragX,this.creature.dragY);
      const stretch = gsap.utils.clamp(0,1,gsap.utils.mapRange(0,stageSize.h*0.5,0,1,d));
      gsap.set(this.creature.$el,{ rotation:a, scaleX:1+stretch*2, scaleY:1-stretch*0.25 });
      if(stretch===1){
        this.creature.setState(CreatureStates.dragging);
        for(let i=0;i<20;i++) spawnParticle({
          className:'ground-particle',
          startX:this.creature.startX+gsap.utils.random(-this.creature.radius*0.5,this.creature.radius*0.5),
          startY:this.creature.startY,
          scale:gsap.utils.random(0.25,1),
          velocity:gsap.utils.random(400,800),
          angle:a+gsap.utils.random(-40,40),
          gravity:1200,
          duration:gsap.utils.random(0.5,2)
        });
      }
    }
  }

  class CreatureDraggingState extends CreatureState{
    onEnter(){ gsap.ticker.add(this.tick); this.lockStretch=true; this.transition=gsap.to(this.creature.$el,{scaleX:1, scaleY:1, ease:'elastic.out', duration:1}); }
    onExit(){ gsap.ticker.remove(this.tick); }
    tick=()=>{
      const {deltaX,deltaY,x,y}=this.creature.draggable;
      const l=length(deltaX,deltaY);
      this.creature.qX(x); this.creature.qY(y);
      if(l>20) { this.transition.kill(); this.lockStretch=false; }
      if(this.lockStretch) return;
      const a=angle(0,0,deltaX,deltaY);
      const stretch=gsap.utils.clamp(0,1,gsap.utils.mapRange(0,50,0,1,l));
      gsap.set(this.creature.$el,{rotation:a, scaleX:1+stretch*0.5, scaleY:1-stretch*0.125});
    }
  }

  class CreatureDroppingState extends CreatureState{
    onEnter(){
      this.creature.draggable.disable();
      const tl=gsap.timeline({onComplete:()=>this.creature.setState(CreatureStates.leaving)});
      const d=stageSize.h-this.creature.dragY;
      const duration=d*0.002;
      const squish=gsap.utils.mapRange(0,stageSize.h,0.25,1,d);
      tl.set(this.creature.$el,{zIndex:1});
      tl.to(this.creature.$el,{rotation:0, scaleX:1, scaleY:1, duration:duration*0.5},0);
      tl.to(this.creature.$el,{y:stageSize.h-this.creature.radius,ease:'power3.in',duration:duration},0);
      tl.add(()=>{ const count=gsap.utils.mapRange(0,stageSize.h,4,20,d)|0;
        for(let i=0;i<count;i++) spawnParticle({
          className:'ground-particle',
          startX:this.creature.dragX,
          startY:this.creature.startY,
          scale:gsap.utils.random(0.25,1),
          velocity:gsap.utils.random(100,300),
          angle:-90+gsap.utils.random(-30,30),
          gravity:1200,
          duration:gsap.utils.random(1,4)
        }); });
      tl.set(this.creature.$el,{transformOrigin:'50% 100%'});
      tl.to(this.creature.$el,{scaleY:1-0.75*squish, scaleX:1+0.5*squish, duration:0.25, ease:'expo.out'});
      tl.to(this.creature.$el,{scaleY:1, scaleX:1, duration:1, ease:'elastic.out'});
      tl.set(this.creature.$el,{transformOrigin:'50% 50%'});
    }
  }

  class CreatureLeavingState extends CreatureState{
    onEnter(){
      const tl=gsap.timeline({onComplete:this.creature.handleComplete});
      const legs=Array.from(this.creature.$el.querySelectorAll('.leg'));
      const body=this.creature.$el;
      const gait=((this.creature.radius+this.creature.leg)*Math.PI*2)/8;
      const dir=gsap.utils.random([-1,1]);
      const steps=Math.ceil(dir===1?this.creature.dragX/gait:(stageSize.w-this.creature.dragX)/gait)+1;
      tl.set(body,{scaleX:dir,rotation:0});
      tl.to(body,{y:stageSize.h-(this.creature.radius+this.creature.leg),duration:.5,ease:'back.out(3)'},0);
      tl.to(legs,{y:this.creature.leg*0.5+this.creature.radius,duration:.125,ease:'expo.out'},0);
      tl.to(legs[0],{rotation:'+=45',duration:1});
      const step=(even)=>{ tl.to(body,{rotation:dir===1?'-=45':'+=45',x:(dir===1?'-=':'+=')+gait,duration:0.25,ease:'circ.inOut'},'-=0.25'); tl.to(legs[even?1:0],{rotation:'+=90',duration:0.5,ease:'back.out'}); }
      for(let i=0;i<steps;i++) step(i%2===0);
    }
  }

  class Creature{
    previousState=null; state=CreatureStates.spawning; startX=0; startY=0; width=80; height=80; leg=40;
    constructor(x,y,color,size,leg,onComplete){
      this.$group=createGroup({color,size,leg});
      $stage.appendChild(this.$group);
      this.$dragger=this.$group.querySelector('.dragger');
      this.$el=this.$group.querySelector('.creature');
      this.onComplete=onComplete;
      this.startX=x; this.startY=y; this.width=size; this.height=size; this.leg=leg; this.radius=size*0.5;
      gsap.set([this.$dragger,this.$el],{xPercent:-50,yPercent:-50,x:this.startX,y:this.startY});
      this.qX=gsap.quickTo(this.$el,'x',{duration:0.2,ease:'back.out'});
      this.qY=gsap.quickTo(this.$el,'y',{duration:0.2,ease:'back.out'});
      this.draggable=Draggable.create(this.$dragger,{
        bounds:{top:0,left:0,width:stageSize.w,height:stageSize.h+this.radius},
        onDragStart:this.onDragStart,
        onDragEnd:this.onDragEnd
      })[0];
      this.states={
        [CreatureStates.idle]:new CreatureIdleState(this),
        [CreatureStates.pulling]:new CreaturePullingState(this),
        [CreatureStates.dragging]:new CreatureDraggingState(this),
        [CreatureStates.dropping]:new CreatureDroppingState(this),
        [CreatureStates.leaving]:new CreatureLeavingState(this)
      };
      this.setState(CreatureStates.idle);
    }
    setState(state){
      const prev=this.states[this.state];
      const next=this.states[state];
      if(prev) prev.onExit(state);
      if(next) next.onEnter(this.state);
      this.previousState=this.state; this.state=state;
    }
    onDragStart=()=>{ this.setState(CreatureStates.pulling); }
    onDragEnd=()=>{ this.state===CreatureStates.dragging?this.setState(CreatureStates.dropping):this.setState(CreatureStates.idle); }
    handleComplete=()=>{ this.destroy(); this.onComplete(); }
    destroy(){ this.draggable.kill(); this.$group.remove(); }
    get dragX(){ return this.draggable.x; }
    get dragY(){ return this.draggable.y; }
  }

  // spawn loop ديناميكي
  let creatureCount = 0;
  const colors = ['gold','salmon','lightpink','coral','violet','slateblue'];
  const spawnLoop = () => {
    const color = colors[Math.floor(Math.random()*colors.length)];
    const size = gsap.utils.random(40,180,1);
    const leg = size*gsap.utils.random(0.1,0.8,0.1);
    const startX = gsap.utils.random(100, stageSize.w-100);
    creatureCount++;
    new Creature(startX, stageSize.h, color, size, leg, ()=>{ creatureCount--; if(creatureCount<5) spawnLoop(); });
  }

  for(let i=0;i<5;i++) spawnLoop();

});
</script>

<style>
.stage {
  width: 100%;
  height: 100vh;
  position: relative;
  overflow: hidden;
  background: #111;
}
.group {
  position: absolute;
}
.creature {
  width: var(--size);
  height: var(--size);
  background-color: var(--color);
  border-radius: 50%;
  position: relative;
}
.leg {
  width: calc(var(--leg)/2);
  height: var(--leg);
  background: var(--color);
  position: absolute;
  bottom: 0;
  left: 0;
}
.dragger {
  width: 40px;
  height: 40px;
  position: absolute;
  top:50%;
  left:50%;
  transform: translate(-50%,-50%);
  cursor: grab;
}
.snooze-particle {
  color:white;
  font-weight:bold;
  position:absolute;
}
.ground-particle {
  background:white;
  width:4px;
  height:4px;
  border-radius:50%;
  position:absolute;
}
</style>

<div class="stage"></div> -->


<!-- <script>
  import './app.css';
  import { onMount } from "svelte";
  import { gsap } from "gsap";
  import { ScrollSmoother } from "gsap/ScrollSmoother";

  gsap.registerPlugin(ScrollSmoother);

  onMount(() => {
    // إعداد skew سريع للصور
    const skewSetter = gsap.quickTo(".images img", "skewY", { duration: 0.3 });
    const clamp = gsap.utils.clamp(-20, 20);

    ScrollSmoother.create({
      wrapper: "#wrapper",
      content: "#content",
      smooth: 2,
      effects: true,
      onUpdate: self => {
        skewSetter(clamp(self.getVelocity() / -50));
      },
      onStop: () => skewSetter(0)
    });
  });
</script>

<div id="wrapper">
  <section id="content">
    <h1 class="text">Scrolly Images</h1>
    <h1 aria-hidden="true" class="text outline-text">Scrolly Images</h1>
    <h1 aria-hidden="true" class="text filter-text">Scrolly Images</h1>

    <section class="images">
      <img data-speed="0.8" src="https://images.unsplash.com/photo-1556856425-366d6618905d?ixid=MnwxMjA3fDB8MHxzZWFyY2h8MTV8fG5lb258ZW58MHx8MHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=400&q=60" alt="">
      <img data-speed="0.9" src="https://images.unsplash.com/photo-1520271348391-049dd132bb7c?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=400&q=80" alt="">
      <img data-speed="1" src="https://images.unsplash.com/photo-1609166214994-502d326bafee?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=400&q=80" alt="">
      <img data-speed="1.1" src="https://images.unsplash.com/photo-1589882265634-84f7eb9a3414?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=434&q=80" alt="">
      <img data-speed="0.9" src="https://images.unsplash.com/photo-1514689832698-319d3bcac5d5?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=434&q=80" alt="">
      <img data-speed="1.2" src="https://images.unsplash.com/photo-1535207010348-71e47296838a?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=300&q=80" alt="">
      <img data-speed="0.8" src="https://images.unsplash.com/photo-1588007375246-3ee823ef4851?ixid=MnwxMjA3fDB8MHxzZWFyY2h8MjR8fG5lb258ZW58MHx8MHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=400&q=60" alt="">
      <img data-speed="1" src="https://images.unsplash.com/photo-1571450669798-fcb4c543f6a4?ixid=MnwxMjA3fDB8MHxzZWFyY2h8MjF8fG5lb258ZW58MHx8MHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=400&q=60" alt="">
    </section>
  </section>
</div> -->



<script>
  import './app.css';
import { gsap } from "gsap";
import { SplitText } from "gsap/SplitText";

  gsap.registerPlugin(SplitText);

console.clear();

document.fonts.ready.then(() => {
  gsap.set(".split", { opacity: 1 });

  let split;
  SplitText.create(".split", {
    type: "words,lines",
    linesClass: "line",
    autoSplit: true,
    mask: "lines",
    onSplit: (self) => {
      split = gsap.from(self.lines, {
        duration: 0.6,
        yPercent: 100,
        opacity: 0,
        stagger: 0.1,
        ease: "expo.out",
      });
      return split;
    }
  });

  document.querySelector("button").addEventListener("click", (e) => {
    split.timeScale(0.2).play(0);
  });
});

</script>


<div class="containy">
  <h1 class="split">The text in this paragraph is split by words and lines. We have enabled masking on the lines so that we can animate the lines to create a fun 'reveal' animation. Nice and easy!</h1>
</div>


<style>
  @font-face {
  font-display: block;
  font-family: Mori;
  font-style: normal;
  font-weight: 900;
  src: url(https://assets.codepen.io/16327/PPMori-SemiBold.woff) format("woff");
}

html, body {
  margin:0;
  padding:0;
  width:100%;
  height:100vh;
}
  
body {
  display:flex;
  align-items:center;
  justify-content:center;
  flex-direction: column;
  font-family: "Mori";
/*   background: #0e100f; */
  background: radial-gradient(129% 99% at 112% 85%, rgb(223, 220, 255) 20%, rgb(166, 158, 255) 90%),    
    url('https://assets.codepen.io/16327/noise-e82662fe.png');  
  background-blend-mode: color-dodge;
}

.container {
  max-width: 80vw;
}

.split {
  opacity: 0;
  text-align:center;
  color: rgb(14, 16, 15);
  font-size: clamp(2rem, 5rem, 3vw);
  letter-spacing: 0.05rem;
  will-change: transform;
  color: #0e100f;
}

.split * {
  will-change: transform;
}

button {
  display: inline-block;
  outline: none;
  padding: 8px 14px;
  background: transparent;
  border: solid 4px #0e100f;
  color: #0e100f;
  text-decoration: none;
  border-radius: 99px;
  padding: 12px 25px;
  text-transform: uppercase;
  font-weight: 600;
  cursor: pointer;
  line-height: 18px;
}
</style>