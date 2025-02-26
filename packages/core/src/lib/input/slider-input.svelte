<!--
@component

For numeric user inputs that require easy adjustment.

```svelte
<SliderInput min={0} max={1} step={0.1}  on:input={onInput} />
```
-->
<svelte:options immutable />

<script lang="ts">
import { createEventDispatcher, tick } from 'svelte';
import cx from 'classnames';
import { Tooltip } from '$lib/tooltip';
import NumericInput from './numeric-input.svelte';
import {
  getDecimals,
  parseNumericInputValue,
  type NumericInputTypes,
} from './utils';
import { preventHandler } from '$lib/prevent-handler';

/** The input type */
export let type: NumericInputTypes | undefined = 'number';

/** The value of the input, if any. */
export let value: number | undefined = undefined;

/** The amount to increment/decrement when sliding. */
export let step = 1;

/** The minimum allowed value, if any. */
export let min = Number.NEGATIVE_INFINITY;

/** The maximum allowed value, if any. */
export let max = Number.POSITIVE_INFINITY;

/** Whether or not the input should be rendered as readonly and be operable. */
export let readonly = false;

/** Whether or not the input should be rendered as readonly and be non-operable. */
export let disabled = false;

/** The HTML input element. */
export let input: HTMLInputElement | undefined = undefined;

/** Additional CSS classes to pass to the input container. */
let extraClasses: cx.Argument = '';
export { extraClasses as cx };

const dispatch = createEventDispatcher<{
  /** Fired when the value of the input is edited or the slider is moved. */
  input: number;

  /** Fired when the value of the input is edited or the slider is released. */
  change: number | undefined;
}>();

let numberDragCord: HTMLDivElement;
let numberDragHead: HTMLDivElement;
let isDragging = false;
let startX = 0;
let startValue = 0;

let stepDecimalDigits = 0;
$: {
  const [, decimal = ''] = String(step).split('.');
  stepDecimalDigits = decimal.length;
}

$: isNumber = type === 'number';
$: isButtonDisabled = readonly || disabled;
$: handleDisabled = preventHandler(isButtonDisabled);

$: handlePointerMove = (event: PointerEvent) => {
  const x = event.clientX;
  const deltaString = ((-(startX - x) * step) / 10).toFixed(
    isNumber ? stepDecimalDigits : 0
  );

  const inputValue = input?.value;
  const delta = parseNumericInputValue(deltaString, type);
  const next = parseNumericInputValue(
    (startValue + delta * step).toFixed(getDecimals(inputValue)),
    type
  );

  if (next > max) {
    value = max;
    return;
  }

  if (next < min) {
    value = min;
    return;
  }

  if (next >= startValue) {
    const dx = x - startX;
    numberDragHead.style.transform = `translate(${dx}px, 0px)`;
    numberDragCord.style.cssText = `width: ${dx}px;`;
  } else if (next < startValue) {
    const dx = startX - x;
    numberDragHead.style.transform = `translate(-${dx}px, 0px)`;
    numberDragCord.style.cssText = `
      width: ${dx}px;
      transform: translate(-${dx}px, 0);
    `;
  }

  if (value !== next) {
    value = next;
    dispatch('input', value);
  }
};

const handlePointerUp = () => {
  isDragging = false;
  dispatch('change', value);
};

const handlePointerDown = async (event: PointerEvent) => {
  startX = event.clientX;
  value ||= 0;
  startValue = value;
  isDragging = true;

  input?.focus();

  await tick();

  numberDragHead.style.transform = 'translate(0px, 0px)';
};

const handleInput = () => {
  const next = parseNumericInputValue(input?.value, type);
  value = next;
};

const handleChange = () => {
  const next = parseNumericInputValue(input?.value, type);

  value = next;

  dispatch('input', next);
  dispatch('change', next);
};
</script>

<svelte:document
  on:pointermove={isDragging ? handlePointerMove : undefined}
  on:pointerup={isDragging ? handlePointerUp : undefined}
/>

<div class={cx('relative w-full', extraClasses)}>
  <NumericInput
    {type}
    {step}
    {disabled}
    {readonly}
    {value}
    {...$$restProps}
    cx="pl-3"
    bind:input
    on:blur
    on:keydown
    on:input={handleInput}
    on:change={handleChange}
  />
  <button
    aria-hidden="true"
    disabled={isButtonDisabled}
    tabindex="-1"
    class={cx(
      'absolute bottom-[3px] left-[0.2rem] h-[24px] w-1 cursor-ew-resize',
      {
        'bg-gray-400 hover:bg-gray-700': !isButtonDisabled,
        'cursor-not-allowed bg-disabled-dark': isButtonDisabled,
      }
    )}
    on:pointerdown|preventDefault|stopPropagation={handlePointerDown}
    on:pointerdown|capture={handleDisabled}
  >
    {#if isDragging}
      <div
        bind:this={numberDragCord}
        class="pointer-events-none relative z-max mt-[6px] h-px bg-gray-400"
      />
      <div
        bind:this={numberDragHead}
        class="pointer-events-none relative z-max -ml-[2px] -mt-[5px]"
      >
        <div class="h-2 w-2">
          <Tooltip state="visible">
            <div class="h-2 w-2 rounded-full bg-gray-800" />
            <span slot="description">{value}</span>
          </Tooltip>
        </div>
      </div>
    {/if}
  </button>
</div>
