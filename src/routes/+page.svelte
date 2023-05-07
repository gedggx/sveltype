<script lang="ts">
  import { tweened } from 'svelte/motion'
  import { onMount } from 'svelte'
  import { tick } from 'svelte';

  let Game = 'waiting for input' | 'in progress' | 'paused' | 'game over'
  let Word = String

  let game: Game = 'waiting for input'
  let paused = false
  let seconds = 30
  let typedLetter = ''

  let words: Word[] = []

  let wordIndex = 0
  let letterIndex = 0
  let correctLetters = 0
  let isWordCompleted = 0

  let wordsPerMinute = tweened(0, {delay: 300, duration: 1000})
  let accuracy = tweened(0, {delay: 1300, duration: 1000})

  let wordsEl: HTMLDivElement
  let letterEl: HTMLSpanElement
  let inputEl: HTMLInputElement
  let caretEl: HTMLDivElement

  function getWordsPerMinute() {
    const word = 5
    const minutes = 0.5
    return Math.floor(correctLetters / word / minutes)
  }

  function getAccuracy() {
    const totalLetters = getTotalLetters(words)
    console.debug(totalLetters)
    return Math.floor(correctLetters / totalLetters * 100)
  }

  function getTotalLetters(words: Word[]) {
    return words.reduce((count, word) => count + word.length , 0)
  }

  function getResults() {
    $wordsPerMinute = getWordsPerMinute()
    $accuracy = getAccuracy()
  }

  function updateGameState() {
    setLetter()
    checkLetter()
    nextLetter()
    updateLine()
    resetLetter()
    moveCaret()
  }

  function setLetter() {
    isWordCompleted = letterIndex > (words[wordIndex].length - 1)

    if(!isWordCompleted) {
      letterEl = wordsEl.children[wordIndex].children[letterIndex] as HTMLSpanElement
    }
  }

  function checkLetter() {
    const currentLetter = words[wordIndex][letterIndex]

    if(typedLetter == currentLetter) {
      letterEl.dataset.letter = 'correct'
      increaseScore()
    }

    if(typedLetter != currentLetter) {
      letterEl.dataset.letter = 'incorrect'
    }
  }

  function nextLetter() {
    letterIndex += 1
  }

  function nextWord () {
    const isNotFirstLetter = letterIndex != 0
    const isOneLetterWord = words[wordIndex].length === 1

    if(isNotFirstLetter || isOneLetterWord) {
      let wordEl = wordsEl.children[wordIndex] as HTMLSpanElement
      if(letterIndex < (words[wordIndex].length )) {
        wordEl.dataset.state = "skipped"
      }
      else {
        wordEl.dataset.state = "complete"
      }
      console.debug(wordEl)
      wordIndex += 1
      letterIndex = 0
      increaseScore()
      moveCaret();
    }
  }

  function updateLine() {
    const wordEl = wordsEl.children[wordIndex]
    const wordsY = wordsEl.getBoundingClientRect().y
    const wordY = wordEl.getBoundingClientRect().y

    if(wordY > wordsY) {
      wordEl.scrollIntoView({block: 'center'})
    }
  }

  function resetLetter() {
    typedLetter = ''
  }

  function moveCaret() {
    setLetter();
    const {offsetLeft, offsetTop, offsetWidth} = letterEl

    caretEl.style.top = `${offsetTop}px`
    caretEl.style.left = `${offsetLeft}px`
    if(isWordCompleted) {
      caretEl.style.left = `${offsetLeft+offsetWidth}px`
    }

    caretEl.classList.add('resetNoAnimate')
  }

  function increaseScore() {
    correctLetters += 1
    console.dir(correctLetters);
  }

  function startGame() {
    setGameState('in progress')
    setGameTimer()
  }

  function setGameTimer() {
    function gameTimer() {
      if (seconds > 0 && !paused) {
        seconds -= 1
      }

      if(seconds === 0) {
        clearInterval(interval)
        setGameState('game over')
        getResults()
        resumeGame()
      }

    }
    const interval = setInterval(gameTimer, 1000)
  }

  function setGameState(state: Game) {
    game = state
  }

  function handleKeydown(event: KeyboardEvent) {
    if (event.code == 'Space') {
      event.preventDefault()

      if(game == "in progress") {
        nextWord()
      }
    }

    if (event.key == "Backspace") {
      if(letterIndex > 0) {
        letterIndex -= 1;
        setLetter()
        moveCaret()
        letterEl.dataset.letter = ''
      }
    }
    
    if(game == 'waiting for input') {
      const regex = /^[a-zA-Z ]$/
      if(regex.test(event.key)) {
        startGame()
      }
    }
  }
  
  async function getWords(limit: number) {
    const response = await fetch(`/api/words?limit=${limit}`)
    words = await response.json()
  }

  function focusInput() {
    inputEl.focus()
  }

  function pauseGame() {
    paused = true
  }

  function resumeGame() {
    paused = false
    focusInput()
  }

  onMount(async () => {
    getWords(100)
    focusInput()
  })

</script>

{#if game !== 'game over'}

  <div class="game" data-game={game}>
    <input 
      bind:this={inputEl}
      bind:value={typedLetter}
      on:input={updateGameState}
      on:blur={pauseGame}
      on:keydown={handleKeydown}
      class="input"
      type="text"
    >

    <div class="time">{seconds}</div>

    <div bind:this={wordsEl} class="words">
      {#each words as word}
        <span class="word">
          {#each word as letter} 
            <span class="letter">{letter}</span>
          {/each}
          </span>
        {/each}
        <div bind:this={caretEl} class="caret"></div>
    </div>
  </div>


  {#if paused}

    <div class = "paused" on:click={resumeGame}>
      <div class="message">
        Click to resume...
      </div>
    </div>

  {/if}

{/if}



{#if game === 'game over'}
  <div class="results">
    <div>
      <p class="title">wpm</p>
      <p class="score">{Math.trunc($wordsPerMinute)}</p>
    </div>

    <div>
      <p class = "title">accuracy</p>
      <p class = "score">{Math.trunc($accuracy)}%</p>
    </div>
  </div>
{/if}

<style lang="scss">

  .game {
    position: relative;

    .input {
      position: absolute;
      opacity: 0;
    }

    .time {
      position: absolute;
      top: -48px;
      font-size: 1.5rem;
      color: var(--primary);
      opacity: 0;
      transition: all 0.3s ease;
    }

    &[data-game='in progress'] .time {
      opacity: 1;
    } 

  }


  .words {
    --line-height: 1em;
    --lines: 3;

    width: 100%;
    max-height: calc(var(--line-height) * var(--lines) * 1.42);
    display: flex;
    flex-wrap: wrap;
    gap: 0.6em;
    position: relative;
    font-size: 1.5rem;
    line-height: var(--line-height);
    overflow: hidden;
    user-select: none;
  }

  .letter {
    opacity: 0.4;
    transition: all 0.3s ease;
  }

  :global(.letter[data-letter='correct']) { 
    opacity: 0.8!important;
  }

  :global(.letter[data-letter='incorrect']) { 
    color: var(--primary);
    opacity: 0.8!important;
  }

  :global(.word[data-state='skipped']) { 
    border-bottom: 2px solid red;
  }

  .words {

    .caret {
      position: absolute;
      height: 1.8rem;
      top: 0;
      border-right: 3px solid var(--primary);

      // animation: caret 1s infinite;
      transition: all 0.2s ease;

      @keyframes caret {
        0%,
        to {
          opacity: 0;
        }
        50% {
          opacity: 1;
        }
      }
    }
  }

  .results {
    .title {
      font-size: 2rem;
      color: var(--fg-200);
    }

    .score {
      font-size: 4rem;
      color: var(--primary);
    }

    .play {
      margin-top: 1rem;
    }
  }

  .paused {
    cursor: pointer;
    height: 100%; 
    width: 100%;
    top: 0;
    left: 0;
    position: absolute;
    display: grid;
    align-items: center;
    background: rgb(28 33 43 / 95%);
    text-transform: uppercase;

    .message {
      padding: 20px;
      text-align: center;
      font-size: 32px;
    }
  }

</style>

