---
theme: blood
---
%%
# Description:
Sources for in video images:
- https://gitlab.haskell.org/ghc/ghc-wasm-meta
- https://wiki.haskell.org/Haskell_logos

%%
<style>
:root {--r-code-font: "FiraCode Nerd Font";}
.reveal .hljs {min-height: 50%;}
</style>

## Haskell goes Web

_(Building a frontend-framework in Haskell)_

notes:
Hey, this is Viko, welcome back...  Let's build a new Frontend framework in Haskell.

---

![[HaskellLogoStyPreview-1.png]]

(Haskell logo due to deep Haskell talk)

notes:

# INTRO

You may now ask:
- Why another Frontend Framework?
- Why use Haskell for the Web?
- Why use Haskell at all?

But these are all the wrong questions entirely. There is a simple answer: Just for fun! Or if you want the truth.. I dream about Monads at night...

---

## Haskell ftw
```Haskell 
fib = 1 : 1 : [a + b | (a, b) <- zip fib (tail fib)]
```

notes:

Anyways, so why Haskell for real? Have you seen something this expressive like ever? No? You can't even read what's happening? Well.. in this case just check out one of my older videos about why this computes the Fibonacci sequence (fast, whispering). Anyways, that's besides the point. What i want to say is that I just really like the language and the challenge associated with porting it to the web.

---

## Video & Source Code

[github.com/vvcaw/vids/](https://github.com/vvcaw/vids/)

notes:
Everything you see in this video from the slides the script and the source code of the end product can be found openly available on github. The links can be found in the description below. (And yes I got inspired by noboilerplate's fast technical videos, which you can also find in the description below). And now let's finally get started!

---

```Haskell
mySite = 
	leftContainer <=> rightContainer
	<>
	button "Click Me!"
```

![[layouting.svg|800]]

notes:
So what is the ultimate goal here? We want to have something like this that works in pure Haskell. Therefore it should fullfill the following points:
- There should be an easy and descriptive way of creating the UI
- There should be an easy way of callback functions for button clicks for example
- This also implies that it supports input handling (duh...)
- It should be blazingly fast!!!

---

![[canvas meme.png]]

notes:
Also the whole thing should not use HTML but rather render to canvas to provide optimal performance. This is also the approach that flutter takes and this splits it loose from CSS & HTML completely allowing us to define our own custom layouting system as we please.

That's just a lot of stuff, therefore today we'll only talk about drawing to the HTML canvas, which is already gonna be quite the ride, so buckle up.

---

What we need: Version of GHC that compiles Haskell to WASM
![[ghc-wasm-repo.png]]

notes:
So the first step is compiling Haskell to WASM. For that we need a version of the "glorious" Haskell Compiler that compiles Haskell source code into a WASM reactor module. Luckily there is exactly that and you can find a lot of information and documentation about the whole thing in this repo.

---

### Using Nix
```Nix[1-13]
{
  inputs = {
    nixpkgs.url = "nixpkgs/nixos-23.05";
	# Pull in custom GHC version for WASM
	wasi-ghc = {
      url = "git+https://gitlab.haskell.org/ghc/ghc-wasm-
      meta.git";
      inputs = { };
    };
...
	# GHC version is now available in our flake
	wasi-ghc.packages.x86_64-linux.wasm32-wasi-cabal-gmp
}
```

notes:
As a build system I chose nix, because  `ghc-wasm-meta` literally exposes a nix flake that we can just pull to get the GHC version, which can be seen above. Nix is a big topic and if you are interested in it, just let me know in the comments below. I will certainly make a video about it in the future! Here we fetch the `cabal` wrapper of the GHC version which allows us to use the `cabal` build system for Haskell to build our project.

---

```yaml[1-5]
  c-sources:        cbits/init.c

  ghc-options:
    -no-hs-main -optl-mexec-model=reactor
    "-optl-Wl,--export=hs_init,--export=mallocBytes"
```

```Haskell[1-5]
import qualified Foreign.Marshal.Alloc as A

foreign export ccall mallocBytes :: Int -> IO (Ptr Word8)
mallocBytes :: Int -> IO (Ptr Word8)
mallocBytes = A.mallocBytes
```

notes:
Now let's have a quick look at the .cabal file that specifies which functions we export. Here we can see that we include some C sources in our project that are just used to initialize the Haskell RTS & Garbage collection. Wizer will be used to make sure that before loading the WASM module these configurations will be run. Below we can see that we export a WASM reactor module which is basically just a library in WASM terms. We also define which functions we export. The second codeblock shows the definition and declaration of the exported `mallocBytes` function that will be used to allocate memory on the WASM-side. Notice that we export `mallocBytes` as a C function here.


---

### The build script
```bash
#!/usr/bin/env bash
WDIR="."

wasm32-wasi-cabal build exe:velvet
wasm32-wasi-cabal list-bin exe:velvet
VELVET_WASM="$(wasm32-wasi-cabal list-bin exe:velvet)"

wizer \
    --allow-wasi --wasm-bulk-memory true \
    "$VELVET_WASM" -o "$WDIR/velvet-init.wasm"

mv "$VELVET_WASM" src/velvet.wasm
```

notes:
Let's have a quick look at the build script, though it is incredibly simple. First we use the custom GHC version to compile our project to WASM. Then we run `wizer` to make sure that the Haskell Runtime system and garbage collector get initialized correctly. Finally, we just move our library to the Frontend `src` directory. Running this using `nix develop` ensures perfect reproducability under all circumstances. Nix makes sure you can build your project... on any system with Nix installed, thankfully Nix is available on all major platforms, enough, I'm drifting off...

---

```JS[1-13]
const { mallocBytes, memory, draw } = wasm.instance.exports
const offset = mallocBytes(width * height * SIZE_OF_INT);

// Haskell draw call
draw(width, height, Date.now(), offset);

const screen = new Uint8ClampedArray(
    memory.buffer, // using the C memory...
    offset, // at the offset of the malloc call...
    SIZE_OF_INT * width * height
);

// fill canvas with array....
```

```Haskell[1-4]
draw :: Int -> Int -> Int -> Ptr Word8 -> IO ()
draw width height time ptr = do
    sequence_ [poke (plusPtr ptr i :: Ptr Word8) 187 
	    | i <- [0..((4 * width * height) - 1)]]
```

notes:
Here you can see the Javascript code that is actually calling these exported functions.  First we get the exports from the `wasm` instance object, then we're allocating enough bytes to buffer the entire canvas. Afterwards we call draw. Its implementation is shown below. It basically fills the buffer with one static color. Real programmers don't de-reference pointers, real programmers poke their pointers. Anyways, after filling the buffer we construct a `Uint8ClampedArray` using the start pointer as an offset into `memory.buffer`. This is basically the buffer containing all the linear memory, which means all of the memory available to WASM. After that we fill the canvas with the data.

---

### Filling the buffer
![[Pasted image 20230926215706.png]]

notes:
Now let's have a look at the result... Well.... What did you expect, it's a grey rectangle, but it perfectly shows that our proof of concept is working! We were able to draw an "image" from Haskell onto a HTML-Canvas, well that sure was fun... Next time we're gonna get started building a small interface library for actually rendering something.

---

### Postlude (pun intended.)

notes:
So after finishing the work for this video, I realized that it won't be as easy as I thought to get a library that efficiently draws on the canvas into web assembly. The issue here is that when doing that efficiently, we use the GPU (which is not possible in pure WASM). Therefore we would need to use webgpu which basically erradicates the need for WASM and therefore Haskell. Anyways, I did what I wanted & successfully compiled Haskell to WASM and it works well. Therefore I would call this project DONE.

---

# I like
https://excalidraw.com/

notes:
But now.... welcome to the "I like" section where i will quickly talk about technology that I found recently and find useful. This time it's gonna be excalidraw, it's perfect for getting quick sketches out and it allows handwriting and exporting to SVG which is a big one for me. One graphic from this video was made in excalidraw and it was a breeze. Thank you for watching... and until next time.