   <script>
  import './app.css';
  import { gsap } from "gsap";
  import { onMount } from "svelte";
  import { ScrollSmoother } from "gsap/ScrollSmoother";
  import { Draggable, InertiaPlugin, Physics2DPlugin } from "gsap/all";
  import { SplitText } from "gsap/SplitText";


  gsap.registerPlugin(Draggable, InertiaPlugin, Physics2DPlugin, ScrollSmoother, SplitText);


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
      <div class="group absolute inset-0 pointer-events-none" style="--color:${color}; --leg:${leg}px; --size:${size}px;">
        <div class="dragger absolute w-[var(--size)] h-[var(--size)] rounded-full pointer-events-all"></div>
        <div class="creature absolute w-[var(--size)] h-[var(--size)]">
          <div class="leg "></div>
          <div class="leg"></div>
          <div class="body"></div>
        </div>
      </div>`;
    const template = document.createElement('div');
    template.innerHTML = html;
    return template.querySelector('.group');
  }

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

    this.draggable = Draggable.create(this.$dragger, {
  bounds: { top: 0, left: 0, width: stageSize.w, height: stageSize.h + this.radius },
  onDragStart: this.onDragStart,
  onDragEnd: this.onDragEnd,
  allowNativeTouchScrolling: false,   // <<< أهم شيء
  dragClickables: true,               // يخلي اللمس دقيق
  touchScroll: false                  // يمنع السحب العمودي
})[0];


});


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


});



</script>


<main  class=" h-screen">
  <div class="  w-screen flex justify-center algin-center overflow-hidden ">
  
  
<div class="momo-signature-regular absolute 
    top-[20%] left-[20%] 
    sm:top-[20%] sm:left-[25%]
    md:top-[22%] md:left-[25%]
    lg:top-[20%] lg:left-[32%]
    text-4xl sm:text-5xl md:text-6xl lg:text-7xl">
  JoveLine
</div>


<div class="momo-signature-regular absolute 
    top-[32%] left-[28%] 
    sm:top-[30%] sm:left-[35%]
    md:top-[33%] md:left-[42%]
    lg:top-[30%] lg:left-[43%]
    text-3xl sm:text-4xl md:text-6xl lg:text-7xl">
  Programming
</div>


  <div class="  text-white absolute  top-[50%] sm:left-[30%] sm:text-[80%] md:top-[50%] md:left-[30%] md:text-[100%] lg:top-[50%] lg:left-[40%] lg:text-[150%] " >
  create by maryam wassem
</div>
</div>

  <div class="flex justify-center items-center ">
  <div class=" w-screen  relative blur-3xl">
    <div class="blob1 absolute w-[700px] h-[400px] rounded-full opacity-90 bg-linear-to-br from-[#43a4a2] to-[#0a0063] "></div> 
  </div>

  <div class='w-screen  relative blur-3xl '>
    <div class="blob2 absolute w-[500px] h-[500px] rounded-full opacity-90 bg-linear-to-br from-[#70579e94] to-[#5500ff] "></div> 
  </div>
  </div>
  
   </main>   
       <div class=" momo-signature-regular animate-pulse sm:text-[200%] md:text-[300%] lg:text-[400%] text-center text-white transition-all duration-300 ease-in-out "> HOW I AM?</div>

     <div 
  class="momo-signature-regular max-w-3xl mx-auto text-center leading-relaxed  px-4 text-xl sm:text-2xl md:text-3xl  lg:text-4xl">
My Name is Maryam Wassem. Age 22, a graduate of Computer Engineering from the University of Technology, I graduated as fourth in my department, Information Department. I love learning and developing myself a lot.</div>
<br><br>
<br><br>
       <div class=" momo-signature-regular animate-pulse sm:text-[200%] md:text-[250%] lg:text-[400%] text-center text-white transition-all duration-300 ease-in-out "> My Skills</div>
<br><br>
       <div class=" flex justify-center items-center">
        <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[250%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">HTML</span>
        </div>
        <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[250%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">CSS</span>
        </div>
        <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[250%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">JavaScript</span>
        </div>
        </div>

         <div class=" flex justify-center items-center">
        <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[250%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">MYSQL</span>
        </div>
        <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[250%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">Bootstrap</span>
        </div>
        <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[250%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">C#</span>
        </div>
        </div>

        <div class="flex justify-center items-center">
           <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[250%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">Tamplate code</span>
      </div>
      <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[250%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">CapCut</span>
      </div>
      <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[250%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">Canva</span>
      </div>
        </div>

        <div class=" flex justify-center items-center">
           <div>
        <span class="momo-signature-regular text-gray-600 sm:text-[200%] md:text-[300%] lg:text-[400%] m-4 p-3 rounded-full inline-block transition-transform duration-300 ease-in-out hover:scale-110">PhotoShop</span>
      </div>
      </div>
<br><br>
  <div class=" momo-signature-regular animate-pulse sm:text-[200%] md:text-[100%] lg:text-[400%] text-center text-white transition-all duration-300 ease-in-out "> this my certificates</div>
<br>
   <div class=" flex flex-col justify-center items-center  text-white " id="wrapper">
    <div>
      <img src="img/html.png" alt="HTML Logo" class=" w-[50%] m-4 transition-all duration-300 ease-in-out hover:scale-110  ">
      <img src="img/css.png" alt="CSS Logo" class="w-[50%] ml-[20%] ml-[30%] m-4 transition-all duration-300 ease-in-out hover:scale-110">
      <img src="img/js.png" alt="JavaScript Logo" class="w-[50%]  m-4 transition-all duration-300 ease-in-out hover:scale-110">
      <img src="img/mysql.png" alt="MySQL Logo" class="w-[50%] ml-[30%] m-4 transition-all duration-300 ease-in-out hover:scale-110">  
      <img src="img/bootstrap.png" alt="Bootstrap Logo" class="w-[50%]   m-4 transition-all duration-300 ease-in-out hover:scale-110">
    </div>
   </div>

   <br><br>
   <br><br>
  
   <div class="grid grid-cols-1 text-center  mt-[20%]">
    <div class="momo-signature-regular sm:text-[200%] md:text-[300%] lg:text-[400%] animate-pulse">my create device</div>
    <div>
<a href="https://draken29.github.io/finally-web-page/" 
   class="transition-all duration-300 ease-in-out hover:scale-110 hover:text-cyan-700 inline-block">
   Finally Web Page(every device)
</a>
    </div>
    <div>
      <a href="https://draken29.github.io/Dr.Rusl/Russlpage.html" class="transition-all duration-300 ease-in-out hover:scale-110 hover:text-cyan-700 inline-block">Dr.Rusl Web Page(every device)</a>
    </div>
    <div>
      <a href="https://draken29.github.io/Emergency-Project/imen.html" class="transition-all duration-300 ease-in-out hover:scale-110 hover:text-cyan-700 inline-block ">Emergency Project(only comp)</a>
    </div>
    <div>
      <a href="https://draken29.github.io/EmmaStore/newonepage.html" class="transition-all duration-300 ease-in-out hover:scale-110 hover:text-cyan-700 inline-block ">Emma Store(every device)</a>
    </div>
    <div>
      <a href="https://draken29.github.io/newdevelopment-note-book/page.html" class="transition-all duration-300 ease-in-out hover:scale-110 hover:text-cyan-700 inline-block ">Note Book(every device)</a>
    </div>
    <div>
      <a href="https://draken29.github.io/xo-final-update/xo1.html" class="transition-all duration-300 ease-in-out hover:scale-110 hover:text-cyan-700 inline-block ">XO Game(every comp)</a>
    </div>
    <div>
      <a href="https://draken29.github.io/page-store/page1.html" class="transition-all duration-300 ease-in-out hover:scale-110 hover:text-cyan-700 inline-block ">Store Page(every device)</a>
    </div>
   </div>

    <br><br>
    <br><br>

     <div class=" momo-signature-regular animate-pulse sm:text-[200%] md:text-[100%] lg:text-[400%] text-center text-white transition-all duration-300 ease-in-out "> this my edit</div>


   <div class=" flex flex-col justify-center items-center  text-white " id="wrapper">
    <div>
      <img src="img/اعلان مناظر.png" class=" w-[50%] m-4 transition-all duration-300 ease-in-out hover:scale-110 " alt="glasses">
      <img src="img/النتيجة النهائية من اعلان المطعم.png" alt="resturant" class="w-[50%] ml-[20%] ml-[30%] m-4 transition-all duration-300 ease-in-out hover:scale-110">
      <img src="img/مشروع اليوم.jpg" alt="resturant" class="w-[50%]  m-4 transition-all duration-300 ease-in-out hover:scale-110">
      <img src="img/dins.jpg" alt="dintist" class="w-[50%] ml-[30%] m-4 transition-all duration-300 ease-in-out hover:scale-110">  
      <img src="img/drink.jpg" alt="drink" class="w-[50%]   m-4 transition-all duration-300 ease-in-out hover:scale-110">
      <img src="img/sea.jpg" alt="sea" class="w-[50%] ml-[30%] m-4 transition-all duration-300 ease-in-out hover:scale-110">
      <img src="img/shos.jpg" alt="shoes" class="w-[50%]   m-4 transition-all duration-300 ease-in-out hover:scale-110">
    </div>
   </div>

  <br><br>
  <br><br>
  <div class="momo-signature-regular animate-pulse sm:text-[200%] md:text-[250%] lg:text-[300%] text-center transition-all duration-300 ease-in-out">
    My Project Video
  </div>
  <br><br>
  
   <section class="flex  justify-center items-center text-white ">
  <div class="mr-[20%]">
  <video src="video/one.mp4"  controls class="w-auto h-48 sm:h-56 md:h-64 lg:h-150 mt-6 rounded-lg shadow-lg transition-transform duration-300 ease-in-out hover:scale-105"></video>
  </div>
  <div>
  <video src="video/two.mp4"  controls class="w-auto h-48 sm:h-56 md:h-64 lg:h-150 mt-6 rounded-lg shadow-lg transition-transform duration-300 ease-in-out hover:scale-105"></video>
  </div>
</section>

 <section class="flex  justify-center items-center text-white ">
  <div class="mr-[20%]">
  <video src="video/three.mp4"  controls class="w-auto h-48 sm:h-56 md:h-64 lg:h-150 mt-6 rounded-lg shadow-lg transition-transform duration-300 ease-in-out hover:scale-105"></video>
  </div>
  <div>
  <video src="video/four.mp4"  controls class="w-auto h-48 sm:h-56 md:h-64 lg:h-150 mt-6 rounded-lg shadow-lg transition-transform duration-300 ease-in-out hover:scale-105"></video>
  </div>
</section>

  
    <div class=" stage  w-full h-100 absolute bg-gray-900 overflow-hidden " >
      <p class="mt-[10%] ml-[5%]">*click  or Pull circle</p>
    </div>  
 

<style>
@import url('https://fonts.googleapis.com/css2?family=Momo+Signature&display=swap');

.momo-signature-regular {
  font-family: "Momo Signature", cursive;
  font-weight: 400;
  font-style: normal;
}


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


/* .split {
  opacity: 0;
  text-align:center;
  font-size: clamp(2rem, 5rem, 3vw);
  letter-spacing: 0.05rem;
  will-change: transform;
} */

/* .split * {
  will-change: transform;
} */



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
</style>   
