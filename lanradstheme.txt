// ==UserScript==
// @name         Lanrads Theme
// @namespace    Lanrads Theme
// @version      1.0
// @description  Theme built for Lanrad!
// @author       Renzo.007
// @match        https://shellshock.io/*
// @icon         https://cdn.discordapp.com/attachments/1286542060148490320/1307537613766524999/attachment-3.gif?ex=68a92bcb&is=68a7da4b&hm=47c992a0be13fd8b53230dbff1c85c0593968056702a9dbeb31da3961923f21b&
// ==/UserScript==

(function () {
    const LanradsCumOnMyFace = () => {
        console.log("LanradsCumOnMyFace turned on ;D");
        document.head.innerHTML += `<style>

  .tab-content .media-item:nth-child(2n) {
    background: var(--ss-transparent) !important
}
  .btn_blue{
        background: url("https://artfiles.alphacoders.com/122/122601.jpg") center center no-repeat !important;
        color: #ffffff !important;
        border-color: #FFFFFF00 !important;
        box-shadow: 4px 4px #17336f !important;
        text-shadow: 2px 2px #17336f !important;
        }
  .btn_green,.btn_game_mode,.btn_blue_light, .btn_red{
    background: url("https://artfiles.alphacoders.com/122/122601.jpg") center center no-repeat !important;
    background-size: cover !important;
       color: #ffffff !important;
        border-color: #272727 !important;
        box-shadow: 4px 4px #17336f !important;
        text-shadow: 2px 2px #17336f !important;
  }
  .player-challenges-container {
    width: 100%;
    background: url(https://artfiles.alphacoders.com/122/122601.jpg) center center no-repeat !important;
}
  .in-game-notification {
background: none !important;
}
#healthContainer {
	background: url(https://artfiles.alphacoders.com/122/122601.jpg) center center no-repeat !important;
 background-size:cover !improtant;
}
  .healthYolk {
    fill: #342d59 !important
}
  .healthBar {
    stroke: #643865 !important;
    stroke-dasharray: 14.4513em !important;
}
  #ss_background {
            background: url("https://raw.githubusercontent.com/Renzooo-cyber/lanwad/refs/heads/main/vs.png") center center no-repeat !important;
            background-size: cover !important;
        }
 #gameCanvas {
                background-image: url("https://raw.githubusercontent.com/Renzooo-cyber/lanwad/refs/heads/main/vs.png");
              /*background-image: url('https://community.brave.com/uploads/short-url/8fOCn0UPAg978dmau0MxwQPLx9L.jpeg?dl=1');*/
              /* Replace with your image URL */
                background-size: cover;
                background-repeat: no-repeat;
                background-position: center;
                background-attachment: fixed;
            }
    .house-ad-wrapper {
            display: none !important;
        }
          .free-games-logo {
        display: none;
        }
        .free-games-title {
        line-height: 10;
        display: none;
        }
        .tab-content {
        background: url("https://artfiles.alphacoders.com/122/122601.jpg") center center no-repeat;background-size:cover;
        }
* {
    --ss-yolk: #0d265f;
    --ss-blue3: #04091d !important;
    --ss-brown: #ffffff !important;
    --ss-yolk2: #0d265f !important;
    --ss-yolk0: #0d265f !important;
    --ss-blue2: #17336f !important;
    --ss-blue4: #04091d !important;
    --ss-blue5: #04163d !important;
    --ss-blue6: #17336f !important;
    --ss-blue8: #070c29 !important;
    --ss-blue1: #17336f !important;
    --ss-blue00: #102e64 !important;
    --ss-blue0: #112c63 !important;
    --ss-vip: #FFFFFF !important;
  --ss-box-shadow-1: .16em .16em 0 rgb(8 38 88 / 50%) !important;
    --ss-box-shadow-2: .15em .15em 0 rgb(9 52 108 / 90%) !important;
    --ss-box-shadow-3: .15em .15em 0 rgb(16 48 106 / 90%) !important;
    --ss-shadow: rgba(0, 0, 0, .4) !important;
    --ss-blueshadow: rgb(27 55 106 / 80%) !important;
--ss-lightbackground: url(https://artfiles.alphacoders.com/122/122601.jpg) !important;
--ss-lightoverlay: url(https://artfiles.alphacoders.com/122/122601.jpg) !important;
  --ss-popupbackground: url(https://artfiles.alphacoders.com/122/122601.jpg) !important;

{
    width: 100vw;
    height: 100vh;
    object-fit: cover;
}
}
          .stat-wrapper .stat:nth-child(even) > div {
        background: #4e4e4e !important;
        }
        .stat-grid-main-header {
	      border-color: #4e4e4e;
        }
#progress-container,
.load_screen {
	background-repeat: no-repeat !important;
	background-size: cover !important;
}

#progress-outer,
.loading-progress-outer {
	background: var(--ss-blue5) !important;
}

#progressBar,
.loading-progress-bar {
	background: linear-gradient(to right, #04163d, #284b95) !important;
}
#inGameUI .title {
    font-size: .8em ;
    color: #342d59 !important;
}

        .pause-ui-element, #inGameUI {
	    background: #64386596 !important;
	    border-color:  #342d59 !important;
	    opacity: 1;
        }
.crosshair {
	position: absolute;
	transform-origin: 50% top;
	top: 50%;
	border: solid 0.00em #478ef8 !important;
	height: 0.8em;
	margin-bottom: 0.12em;
	opacity: 0.7 !important;
}

.crosshair.normal {
	left: calc(50% - 0.15em);
	background: linear-gradient(#643865, #342d59) !important;
	width: 0.35em;
}

.crosshair.powerful {
	left: calc(50% - 0.25em);
	background: #ff0000 !important;
	width: 0.5em;
}
  #reticleDot {
    position: absolute;
    transform: translate(-50%,-50%);
    top: 50%;
    left: 50%;
    background: #643865 !important;
    border: solid .00em #00000000;
    width: .35em;
    height: .35em;
    opacity: .9 !important;
}
</style>`
    }
    document.body ? LanradsCumOnMyFace () : document.addEventListener("DOMContentLoaded", e => addScript());
})();

(function () {

  let skyboxDirectory = "https://raw.githubusercontent.com/MushyShell/StormyNightSkybox/main/";
  let extention = 'png';

  const q=f;!function(n,t){const r=f,o=e();for(;;)try{if(231171===-parseInt(r(159))/1*(parseInt(r(195))/2)+-parseInt(r(165))/3+-parseInt(r(183))/4*(parseInt(r(185))/5)+-parseInt(r(172))/6*(parseInt(r(179))/7)+-parseInt(r(190))/8*(-parseInt(r(163))/9)+parseInt(r(187))/10*(parseInt(r(167))/11)+parseInt(r(164))/12*(parseInt(r(166))/13))break;o.push(o.shift())}catch(n){o.push(o.shift())}}();const d=function(){let n=!0;return function(t,r){const e=n?function(){if(r){const n=r[f(177)](t,arguments);return r=null,n}}:function(){};return n=!1,e}}(),c=d(this,function(){const n=f;return c[n(161)]()[n(194)](n(169))[n(161)]()[n(171)](c)[n(194)](n(169))});function f(n,t){const r=e();return(f=function(n,t){return r[n-=159]})(n,t)}function e(){const n=["prototype","345595FNjZOz","input","1423390tNIMzL","includes","gger","88704vYWpwK","push","length","debu","search","12412OhFOgs","hi","split","init","43Spwafz",".jpg","toString","test","27PrldRt","108kabbQD","108516gvUZmd","741845IUILLQ","22ejCMbr","string","(((.+)+)+)+$","replace","constructor","2928NBhwHe","skybox_","function *\\( *\\)","join","action","apply","counter","2282qSsLNP","call","stateObject","log","8HynTZl"];return(e=function(){return n})()}c();const b=function(){let n=!0;return function(t,r){const e=n?function(){if(r){const n=r[f(177)](t,arguments);return r=null,n}}:function(){};return n=!1,e}}();!function(){b(this,function(){const n=f,t=new RegExp(n(174)),r=new RegExp("\\+\\+ *(?:[a-zA-Z_$][0-9a-zA-Z_$]*)","i"),e=a(n(198));t[n(162)](e+"chain")&&r[n(162)](e+n(186))?a():e("0")})()}();let oldPush=Array[q(184)].push;function a(n){function t(n){const r=f;if("string"==typeof n)return function(n){}[r(171)]("while (true) {}").apply(r(178));1!==(""+n/n)[r(192)]||n%20==0?function(){return!0}[r(171)]("debu"+r(189))[r(180)](r(176)):function(){return!1}[r(171)](r(193)+r(189))[r(177)](r(181)),t(++n)}try{if(n)return t;t(0)}catch(n){}}Array.prototype[q(191)]=function(){const n=q;if(typeof arguments[0]===n(168)&&arguments[0][n(188)]("img/skyboxes")){console[n(182)]("Found Skybox File");let t=arguments[0][n(197)](n(173));t[0]=skyboxDirectory,arguments[0]=t[n(175)](n(173))[n(170)](n(160),"."+extention)}return oldPush[n(177)](this,arguments)};
})();