<template>
    <div :class="containerClass" @click="onBarClick">
        <span class="p-slider-range" :style="rangeStyle"></span>
        <span v-if="!range" class="p-slider-handle" :style="handleStyle" @touchstart="onDragStart($event)" @touchmove="onDrag($event)" @touchend="onDragEnd($event)" @mousedown="onMouseDown($event)" @keydown="onKeyDown($event)" :tabindex="tabindex"
            role="slider" :aria-valuemin="min" :aria-valuenow="modelValue" :aria-valuemax="max" :aria-labelledby="ariaLabelledby" :aria-label="ariaLabel" :aria-orientation="orientation"></span>
        <span v-if="range" class="p-slider-handle" :style="rangeStartHandleStyle" @touchstart="onDragStart($event, 0)" @touchmove="onDrag($event)" @touchend="onDragEnd($event)" @mousedown="onMouseDown($event, 0)" @keydown="onKeyDown($event)" :tabindex="tabindex"
            role="slider" :aria-valuemin="min" :aria-valuenow="modelValue ? modelValue[0] : null" :aria-valuemax="max" :aria-labelledby="ariaLabelledby" :aria-label="ariaLabel" :aria-orientation="orientation"></span>
        <span v-if="range" class="p-slider-handle" :style="rangeEndHandleStyle" @touchstart="onDragStart($event, 1)" @touchmove="onDrag($event)" @touchend="onDragEnd($event)" @mousedown="onMouseDown($event, 1)" @keydown="onKeyDown($event, 1)" :tabindex="tabindex"
            role="slider" :aria-valuemin="min" :aria-valuenow="modelValue ? modelValue[1] : null" :aria-valuemax="max" :aria-labelledby="ariaLabelledby" :aria-label="ariaLabel" :aria-orientation="orientation"></span>
    </div>
</template>

<script>
import {DomHandler} from 'primevue/utils';

export default {
    name: 'Slider',
    emits: ['update:modelValue', 'change', 'slideend'],
    props: {
        modelValue: [Number,Array],
		min: {
			type: Number,
			default: 0
		},
		max: {
			type: Number,
			default: 100
		},
		orientation: {
			type: String,
			default: 'horizontal'
		},
		step: {
			type: Number,
			default: null
		},
		range: {
			type: Boolean,
			default: false
		},
        disabled: {
			type: Boolean,
			default: false
        },
        tabindex: {
            type: Number,
            default: 0
        },
        'aria-labelledby': {
            type: String,
			default: null
        },
        'aria-label': {
            type: String,
            default: null
        }
    },
    dragging: false,
    handleIndex: null,
    initX: null,
    initY: null,
    barWidth: null,
    barHeight: null,
    dragListener: null,
    dragEndListener: null,
    beforeUnmount() {
        this.unbindDragListeners();
    },
    methods: {
        updateDomData() {
            let rect = this.$el.getBoundingClientRect();
            this.initX = rect.left + DomHandler.getWindowScrollLeft();
            this.initY = rect.top + DomHandler.getWindowScrollTop();
            this.barWidth = this.$el.offsetWidth;
            this.barHeight = this.$el.offsetHeight;
        },
        setValue(event) {
            let handleValue;
            let pageX = event.touches ? event.touches[0].pageX : event.pageX;
            let pageY = event.touches ? event.touches[0].pageY : event.pageY;
            if(this.orientation === 'horizontal')
                handleValue = ((pageX - this.initX) * 100) / (this.barWidth);
            else
                handleValue = (((this.initY + this.barHeight) - pageY) * 100) / (this.barHeight);
            let newValue = (this.max - this.min) * (handleValue / 100) + this.min;
            if (this.step) {
                const oldValue = this.range ? this.modelValue[this.handleIndex] : this.modelValue;
                const diff = (newValue - oldValue);
                if (diff < 0)
                    newValue = oldValue + Math.ceil(newValue / this.step - oldValue / this.step) * this.step;
                else if (diff > 0)
                    newValue = oldValue + Math.floor(newValue / this.step - oldValue / this.step) * this.step;
            }
            else {
                newValue = Math.floor(newValue);
            }
            this.updateModel(event, newValue);
        },
        updateModel(event, value) {
            let newValue = parseFloat(value.toFixed(10));
            let modelValue;

            if (this.range) {
                modelValue = this.modelValue ? [...this.modelValue] : [];

                if (this.handleIndex == 0) {
                    let maxValue = this.modelValue ? this.modelValue[1] : this.max;

                    if (newValue < this.min)
                        newValue = this.min;
                    else if (newValue >= maxValue)
                        newValue = maxValue;

                    modelValue[0] = newValue;
                    modelValue[1] = modelValue[1] || this.max;
                }
                else {
                    let minValue = this.modelValue ? this.modelValue[0] : this.min;
                    if (newValue > this.max)
                        newValue = this.max;
                    else if (newValue <= minValue)
                        newValue = minValue;

                    modelValue[0] = modelValue[0] || this.min;
                    modelValue[1] = newValue;
                }
            }
            else {
                if (newValue < this.min)
                    newValue = this.min;
                else if (newValue > this.max)
                    newValue = this.max;

                modelValue = newValue;
            }

            this.$emit('update:modelValue', modelValue);
            this.$emit('change', modelValue);
        },
        onDragStart(event, index) {
            if (this.disabled) {
                return;
            }

            DomHandler.addClass(this.$el, 'p-slider-sliding');
            this.dragging = true;
            this.updateDomData();

            if (this.range && this.modelValue[0] === this.max) {
                this.handleIndex = 0;
            }
            else {
                this.handleIndex = index;
            }

            event.preventDefault();
        },
        onDrag(event) {
            if (this.dragging) {
                this.setValue(event);
                event.preventDefault();
            }
        },
        onDragEnd(event) {
            if (this.dragging) {
                this.dragging = false;
                DomHandler.removeClass(this.$el, 'p-slider-sliding');
                this.$emit('slideend', {originalEvent: event, value: this.modelValue});
            }
        },
        onBarClick(event) {
            if (this.disabled) {
                return;
            }
            if (!DomHandler.hasClass(event.target, 'p-slider-handle')) {
                this.updateDomData();
                this.setValue(event);
            }
        },
        onMouseDown(event, index) {
            this.bindDragListeners();
            this.onDragStart(event, index);
        },
        onKeyDown(event, index) {
            this.handleIndex = index;
            switch (event.which) {
                //down and left
                case 40:
                case 37:
                    this.decrementValue(event, index);
                    event.preventDefault();
                    
                break;

                //up and right
                case 38:
                case 39:
                    this.incrementValue(event, index);
                    event.preventDefault();
                    
                break;

                default:
                break;
            }
        },
        decrementValue(event, index) {
            let newValue;
            if (this.range) {
                if (this.step)
                    newValue = this.modelValue[index] - this.step;
                else
                    newValue = this.modelValue[index] - 1;
            }
            else {
                if (this.step)
                    newValue = this.modelValue - this.step;
                else
                    newValue = this.modelValue - 1;
            }
            this.updateModel(event, newValue);
            event.preventDefault();
        },
        incrementValue(event, index) {
            let newValue;
            if (this.range) {
                if (this.step)
                    newValue = this.modelValue[index] + this.step;
                else
                    newValue = this.modelValue[index] + 1;
            }
            else {
                if (this.step)
                    newValue = this.modelValue + this.step;
                else
                    newValue = this.modelValue + 1;
            }
            this.updateModel(event, newValue);
            event.preventDefault();
        },
        bindDragListeners() {
            if (!this.dragListener) {
                this.dragListener = this.onDrag.bind(this);
                document.addEventListener('mousemove', this.dragListener);
            }
            if (!this.dragEndListener) {
                this.dragEndListener = this.onDragEnd.bind(this);
                document.addEventListener('mouseup', this.dragEndListener);
            }
        },
        unbindDragListeners() {
            if (this.dragListener) {
                document.removeEventListener('mousemove', this.dragListener);
                this.dragListener = null;
            }
            if (this.dragEndListener) {
                document.removeEventListener('mouseup', this.dragEndListener);
                this.dragEndListener = null;
            }
        }
    },
    computed: {
        containerClass() {
            return ['p-slider p-component', {
                'p-disabled': this.disabled,
                'p-slider-horizontal': (this.orientation === 'horizontal'),
                'p-slider-vertical': (this.orientation === 'vertical')
            }];
        },
        horizontal() {
            return this.orientation === 'horizontal';
        },
        vertical() {
            return this.orientation === 'vertical';
        },
        rangeStyle() {
            if (this.range) {
                if (this.horizontal)
                    return {'left': this.rangeStartPosition + '%', width: (this.rangeEndPosition - this.rangeStartPosition + '%')};
                else
                    return {'bottom': this.rangeStartPosition + '%', height: (this.rangeEndPosition - this.rangeStartHandlePosition + '%')};
            }
            else {
                if (this.horizontal)
                    return {'width': this.handlePosition + '%'};
                else
                    return {'height': this.handlePosition + '%'};
            }
        },
        handleStyle() {
            if (this.horizontal)
                return {'left': this.handlePosition + '%'};
            else
                return {'bottom': this.handlePosition + '%'};
        },
        handlePosition() {
            if (this.modelValue < this.min)
                return 0;
            else if (this.modelValue > this.max)
                return 100;
            else
                return (this.modelValue - this.min) * 100 / (this.max - this.min);
        },
        rangeStartPosition() {
            if (this.modelValue && this.modelValue[0])
                return (this.modelValue[0] < this.min ? 0 : this.modelValue[0] - this.min) * 100 / (this.max - this.min);
            else
                return 0;
        },
        rangeEndPosition() {
            if (this.modelValue && this.modelValue.length === 2)
                return (this.modelValue[1] > this.max ? 100 : this.modelValue[1] - this.min) * 100 / (this.max - this.min);
            else
                return 100;
        },
        rangeStartHandleStyle() {
            if (this.horizontal)
                return {'left': this.rangeStartPosition + '%'};
            else
                return {'bottom': this.rangeStartPosition + '%'};
        },
        rangeEndHandleStyle() {
            if (this.horizontal)
                return {'left': this.rangeEndPosition + '%'};
            else
                return {'bottom': this.rangeEndPosition + '%'};
        },
    }
}
</script>

<style>
.p-slider {
	position: relative;
}

.p-slider .p-slider-handle {
	position: absolute;
	cursor: grab;
    touch-action: none;
    display: block;
}

.p-slider-range {
	position: absolute;
    display: block;
}

.p-slider-horizontal .p-slider-range {
    top: 0;
    left: 0;
	height: 100%;
}

.p-slider-horizontal .p-slider-handle {
    top: 50%;
}

.p-slider-vertical {
	height: 100px;
}

.p-slider-vertical .p-slider-handle {
    left: 50%;
}

.p-slider-vertical .p-slider-range {
    bottom: 0;
    left: 0;
    width: 100%;
}
</style>
