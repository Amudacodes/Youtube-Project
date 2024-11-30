<script lang="ts">
	import { onMount, tick } from 'svelte';
	import { tweened } from 'svelte/motion';
	import { elasticOut, backOut } from 'svelte/easing';

	let value = '';
	let inputEl: HTMLInputElement;
	let container: HTMLDivElement;
	let cursorX = 0,
		cursorY = 0;
	let characters: string[] = [];
	let glitchIntensity = tweened(0);
	let contextLost = false;
	let realityFragments: any[] = [];
	let shockwaves: any[] = [];
	let mousePath: { x: number; y: number }[] = [];
	let frameCount = 0;
	let lastKey = '';
	let explosionParticles: any[] = [];
	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let audioCtx: AudioContext;
	let glitchTimeout: any;

	function trackMouse(e: MouseEvent) {
		cursorX = e.clientX;
		cursorY = e.clientY;
		mousePath = [...mousePath, { x: cursorX, y: cursorY }].slice(-50);
		createShockwave(cursorX, cursorY);
		distortReality();
	}

	function explodeCharacters(char: string, x: number, y: number) {
		const particleCount = 50;
		for (let i = 0; i < particleCount; i++) {
			const angle = (Math.PI * 2 * i) / particleCount;
			const velocity = 5 + Math.random() * 10;
			const life = 1 + Math.random() * 2;
			explosionParticles.push({
				char,
				x,
				y,
				vx: Math.cos(angle) * velocity,
				vy: Math.sin(angle) * velocity,
				life,
				maxLife: life,
				rotation: Math.random() * 360,
				scale: 0.5 + Math.random()
			});
		}
		explosionParticles = explosionParticles;
	}

	function createShockwave(x: number, y: number) {
		shockwaves = [
			...shockwaves,
			{
				x,
				y,
				size: 0,
				maxSize: 200,
				life: 1
			}
		];
	}

	async function distortReality() {
		if (!container || contextLost) return;

		const intensity = value.length * 0.1;
		const distortionX = (cursorX / window.innerWidth - 0.5) * 100;
		const distortionY = (cursorY / window.innerHeight - 0.5) * 100;

		container.style.transform = `
      perspective(${800 + Math.sin(frameCount * 0.01) * 200}px)
      rotateX(${distortionY}deg)
      rotateY(${distortionX}deg)
      scale(${1 + Math.sin(frameCount * 0.05) * 0.1})
    `;

		if (audioCtx) {
			const oscillator = audioCtx.createOscillator();
			const gain = audioCtx.createGain();
			oscillator.connect(gain);
			gain.connect(audioCtx.destination);

			oscillator.frequency.setValueAtTime(200 + intensity * 100, audioCtx.currentTime);
			gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
			gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.1);

			oscillator.start();
			oscillator.stop(audioCtx.currentTime + 0.1);
		}
	}

	async function handleKeyInput(e: KeyboardEvent) {
		lastKey = e.key;
		await glitchIntensity.set(1, { duration: 100 });
		await glitchIntensity.set(0, { duration: 500, easing: elasticOut });

		const rect = inputEl.getBoundingClientRect();
		explodeCharacters(e.key, rect.x + rect.width * Math.random(), rect.y);

		clearTimeout(glitchTimeout);
		contextLost = true;
		glitchTimeout = setTimeout(() => (contextLost = false), 50);

		realityFragments = [
			...realityFragments,
			{
				char: e.key,
				x: Math.random() * window.innerWidth,
				y: Math.random() * window.innerHeight,
				vx: (Math.random() - 0.5) * 10,
				vy: (Math.random() - 0.5) * 10,
				rotation: Math.random() * 360,
				scale: 0.5 + Math.random() * 2
			}
		];
	}

	function animate() {
		if (!ctx) return;
		frameCount++;

		ctx.fillStyle = 'rgba(0, 0, 0, 0.1)';
		ctx.fillRect(0, 0, canvas.width, canvas.height);

		explosionParticles = explosionParticles.filter((p) => {
			p.x += p.vx;
			p.y += p.vy;
			p.life -= 0.016;
			p.rotation += 5;
			return p.life > 0;
		});

		explosionParticles.forEach((p) => {
			const alpha = p.life / p.maxLife;
			ctx.save();
			ctx.translate(p.x, p.y);
			ctx.rotate((p.rotation * Math.PI) / 180);
			ctx.scale(p.scale, p.scale);
			ctx.fillStyle = `hsla(${frameCount % 360}, 100%, 50%, ${alpha})`;
			ctx.font = '20px monospace';
			ctx.fillText(p.char, 0, 0);
			ctx.restore();
		});

		shockwaves = shockwaves.filter((s) => {
			s.size += 10;
			s.life -= 0.02;
			return s.life > 0;
		});

		shockwaves.forEach((s) => {
			ctx.strokeStyle = `hsla(${frameCount % 360}, 100%, 50%, ${s.life})`;
			ctx.lineWidth = 2;
			ctx.beginPath();
			ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
			ctx.stroke();
		});

		// Draw mouse trail
		if (mousePath.length > 1) {
			ctx.beginPath();
			ctx.moveTo(mousePath[0].x, mousePath[0].y);
			for (let i = 1; i < mousePath.length; i++) {
				const p = mousePath[i];
				ctx.lineTo(p.x, p.y);
			}
			ctx.strokeStyle = `hsla(${frameCount % 360}, 100%, 50%, 0.5)`;
			ctx.lineWidth = 4;
			ctx.stroke();
		}

		requestAnimationFrame(animate);
	}

	onMount(async () => {
		canvas.width = window.innerWidth;
		canvas.height = window.innerHeight;
		ctx = canvas.getContext('2d')!;
		audioCtx = new AudioContext();
		animate();

		window.addEventListener('resize', () => {
			canvas.width = window.innerWidth;
			canvas.height = window.innerHeight;
		});
	});
</script>




<div 
  bind:this={container}
  class="reality-container"
  on:mousemove={trackMouse}
  style:--glitch-intensity={$glitchIntensity}
>
  <canvas
    bind:this={canvas}
    class="reality-canvas"
  />

  <div class="input-wrapper" style:transform="scale({1 + $glitchIntensity * 0.2})">
    <input
      bind:this={inputEl}
      bind:value
      on:keydown={handleKeyInput}
      class="reality-input"
      placeholder="DESTROY//REALITY//"
    />
    

    {#each realityFragments as fragment}
      <div
        class="reality-fragment"
        style:left="{fragment.x}px"
        style:top="{fragment.y}px"
        style:transform="
          rotate({fragment.rotation + frameCount}deg)
          scale({fragment.scale})
          translate3d(
            {Math.sin(frameCount * 0.1) * 50}px,
            {Math.cos(frameCount * 0.1) * 50}px,
            {Math.random() * 500}px
          )"
      >
        {fragment.char}
      </div>
    {/each}

    {#if lastKey}
      <div class="last-key-effect">
        {lastKey}
      </div>
    {/if}
  </div>
</div>


<style>
  .reality-container {
    @apply fixed inset-0 overflow-hidden bg-black;
    perspective: 1000px;
    transform-style: preserve-3d;
  }

  .reality-canvas {
    @apply fixed inset-0 pointer-events-none;
    z-index: 1;
  }

  .input-wrapper {
    @apply fixed inset-0 flex items-center justify-center;
    z-index: 2;
  }

  .reality-input {
    @apply w-96 px-8 py-6 text-2xl font-mono;
    background: rgba(0, 0, 0, 0.5);
    border: 2px solid;
    border-image: linear-gradient(
      45deg,
      hsl(calc(var(--glitch-intensity) * 360), 100%, 50%),
      hsl(calc(var(--glitch-intensity) * 720), 100%, 50%)
    ) 1;
    color: hsl(calc(var(--glitch-intensity) * 360), 100%, 80%);
    text-shadow: 0 0 10px currentColor;
    backdrop-filter: blur(10px);
    animation: input-pulse 2s infinite;
  }

  .reality-fragment {
    @apply fixed font-mono text-4xl pointer-events-none;
    color: hsl(calc(var(--glitch-intensity) * 360), 100%, 50%);
    text-shadow: 0 0 20px currentColor;
    animation: fragment-float 4s infinite;
  }

  .last-key-effect {
    @apply fixed font-mono text-9xl pointer-events-none text-center w-full;
    color: hsl(calc(var(--glitch-intensity) * 360), 100%, 50%);
    text-shadow: 0 0 30px currentColor;
    animation: key-pulse 0.5s ease-out;
    opacity: var(--glitch-intensity);
  }

  @keyframes input-pulse {
    0%, 100% {
      filter: hue-rotate(0deg) blur(0px);
      transform: scale(1) rotate(0deg);
    }
    50% {
      filter: hue-rotate(180deg) blur(calc(var(--glitch-intensity) * 10px));
      transform: scale(calc(1 + var(--glitch-intensity) * 0.2)) rotate(calc(var(--glitch-intensity) * 5deg));
    }
  }

  @keyframes fragment-float {
    0%, 100% {
      transform: translate3d(0, 0, 0) rotate(0deg);
      opacity: 1;
    }
    50% {
      transform: translate3d(
        calc(var(--glitch-intensity) * 200px),
        calc(var(--glitch-intensity) * -200px),
        200px
      ) rotate(360deg);
      opacity: 0.5;
    }
  }

  @keyframes key-pulse {
    0% {
      transform: scale(0) rotate(0deg);
      opacity: 1;
    }
    100% {
      transform: scale(2) rotate(20deg);
      opacity: 0;
    }
  }

  :global(body) {
    margin: 0;
    overflow: hidden;
    background: black;
  }
</style>