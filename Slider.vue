<template>
    <div>
        <slot/>
    </div>
</template>

<script>
    /**
     * Siema & vue2-siema merged and adapted.
     */
    export default {
        name: 'Slider',
        components: {},
        props: {
            options: {
                type: Object,
                default: () => ({})
            },
            autoPlay: {
                type: Boolean,
                default: false
            },
            playDuration: {
                type: Number,
                default: 6000
            },
            current: {
                type: Number,
                default: 0
            },
            ready: {
                type: Boolean,
                default: true
            },
            name: {
                type: String,
                default: ''
            }
        },
        data() {
            return {
                el: null,
                config: null,
                playing: this.autoPlay,
                time: this.playDuration,
                selector: null,
            }
        },
        mounted() {
            this.construct(this.options);
        },
        methods: {

            /**
             * Constructor.
             */
            construct(options) {
                this.config = this.mergeSettings(options);// Merge defaults with user's settings

                this.selector = (typeof this.config.selector === 'string') ? document.querySelector(this.config.selector) : this.config.selector;// Resolve selector's type

                if (this.selector === null) {// Early throw if selector doesn't exists
                    throw new Error('Something wrong with your selector 😭');
                }

                this.resolveSlidesNumber();// update perPage number dependable of user value

                this.selectorWidth = this.selector.offsetWidth; // Create global references
                this.innerElements = [].slice.call(this.selector.children);
                this.currentSlide = this.config.loop ?
                    this.config.startIndex % this.innerElements.length :
                    Math.max(0, Math.min(this.config.startIndex, this.innerElements.length - this.perPage));
                this.transformProperty = this.webkitOrNot();

                // Bind all event handlers
                ['resizeHandler', 'touchstartHandler', 'touchendHandler', 'touchmoveHandler', 'mousedownHandler', 'mouseupHandler', 'mouseleaveHandler', 'mousemoveHandler', 'clickHandler'].forEach(method => {
                    this[method] = this[method].bind(this);
                });

                // Build markup and apply required styling to elements
                this.init();
            },

            /**
             * Merge user created settings into default settings.
             */
            mergeSettings(options) {
                const settings = {
                    selector: '.siema',
                    duration: 200,
                    easing: 'ease-out',
                    perPage: 1,
                    startIndex: 0,
                    draggable: true,
                    multipleDrag: true,
                    threshold: 20,
                    loop: false,
                    rtl: false,
                    onInit: () => {
                    },
                    onChange: () => {
                    },
                };

                const userSettings = options;
                for (const attrName in userSettings) {
                    settings[attrName] = userSettings[attrName];
                }

                return settings;
            },

            /**
             * No idea.
             */
            webkitOrNot() {
                const style = document.documentElement.style;
                if (typeof style.transform === 'string') {
                    return 'transform';
                }
                return 'WebkitTransform';
            },

            /**
             * Attaches listeners to required events.
             */
            attachEvents() {
                window.addEventListener('resize', this.resizeHandler);// Resize element on window resize

                if (this.config.draggable) {// If element is draggable, add event handlers
                    this.pointerDown = false;// Keep track pointer hold and dragging distance
                    this.drag = {
                        startX: 0,
                        endX: 0,
                        startY: 0,
                        letItGo: null,
                        preventClick: false,
                    };

                    this.selector.addEventListener('touchstart', this.touchstartHandler);// Touch events
                    this.selector.addEventListener('touchend', this.touchendHandler);
                    this.selector.addEventListener('touchmove', this.touchmoveHandler);


                    this.selector.addEventListener('mousedown', this.mousedownHandler);// Mouse events
                    this.selector.addEventListener('mouseup', this.mouseupHandler);
                    this.selector.addEventListener('mouseleave', this.mouseleaveHandler);
                    this.selector.addEventListener('mousemove', this.mousemoveHandler);


                    this.selector.addEventListener('click', this.clickHandler);// Click
                }
            },

            /**
             * Detaches listeners from required events.
             */
            detachEvents() {
                window.removeEventListener('resize', this.resizeHandler);
                this.selector.removeEventListener('touchstart', this.touchstartHandler);
                this.selector.removeEventListener('touchend', this.touchendHandler);
                this.selector.removeEventListener('touchmove', this.touchmoveHandler);
                this.selector.removeEventListener('mousedown', this.mousedownHandler);
                this.selector.removeEventListener('mouseup', this.mouseupHandler);
                this.selector.removeEventListener('mouseleave', this.mouseleaveHandler);
                this.selector.removeEventListener('mousemove', this.mousemoveHandler);
                this.selector.removeEventListener('click', this.clickHandler);
            },

            /**
             * Builds the markup and attaches listeners to required events.
             */
            init() {
                this.attachEvents();
                this.selector.style.overflow = 'hidden';// hide everything out of selector's boundaries
                this.selector.style.direction = this.config.rtl ? 'rtl' : 'ltr';// rtl or ltr
                this.buildSliderFrame();// build a frame and slide to a currentSlide
                this.config.onInit.call(this);
            },

            /**
             * Build a sliderFrame and slide to a current item.
             */
            buildSliderFrame() {
                const widthItem = this.selectorWidth / this.perPage;
                const itemsToBuild = this.config.loop ? this.innerElements.length + (2 * this.perPage) : this.innerElements.length;


                this.sliderFrame = document.createElement('div');// Create frame and apply styling
                this.sliderFrame.style.width = `${widthItem * itemsToBuild}px`;
                this.enableTransition();

                if (this.config.draggable) {
                    this.selector.style.cursor = '-webkit-grab';
                }


                const docFragment = document.createDocumentFragment();// Create a document fragment to put slides into it


                if (this.config.loop) {// Loop through the slides, add styling and add them to document fragment
                    for (let i = this.innerElements.length - this.perPage; i < this.innerElements.length; i++) {
                        const element = this.buildSliderFrameItem(this.innerElements[i].cloneNode(true));
                        docFragment.appendChild(element);
                    }
                }
                for (let i = 0; i < this.innerElements.length; i++) {
                    const element = this.buildSliderFrameItem(this.innerElements[i]);
                    docFragment.appendChild(element);
                }
                if (this.config.loop) {
                    for (let i = 0; i < this.perPage; i++) {
                        const element = this.buildSliderFrameItem(this.innerElements[i].cloneNode(true));
                        docFragment.appendChild(element);
                    }
                }

                this.sliderFrame.appendChild(docFragment);// Add fragment to the frame

                this.selector.innerHTML = '';// Clear selector (just in case something is there) and insert a frame
                this.selector.appendChild(this.sliderFrame);

                this.slideToCurrent();// Go to currently active slide after initial build
            },

            buildSliderFrameItem(elm) {
                const elementContainer = document.createElement('div');
                elementContainer.style.cssFloat = this.config.rtl ? 'right' : 'left';
                elementContainer.style.float = this.config.rtl ? 'right' : 'left';
                elementContainer.style.width = `${this.config.loop ? 100 / (this.innerElements.length + (this.perPage * 2)) : 100 / (this.innerElements.length)}%`;
                elementContainer.appendChild(elm);
                return elementContainer;
            },

            /**
             * Determines slides number accordingly to clients viewport.
             */
            resolveSlidesNumber() {
                if (typeof this.config.perPage === 'number') {
                    this.perPage = this.config.perPage;
                } else if (typeof this.config.perPage === 'object') {
                    this.perPage = 1;
                    for (const viewport in this.config.perPage) {
                        if (window.innerWidth >= viewport) {
                            this.perPage = this.config.perPage[viewport];
                        }
                    }
                }
            },

            /**
             * Go to previous slide.
             * @param {number} [howManySlides=1] - How many items to slide backward.
             * @param {function} callback - Optional callback function.
             */
            prev(howManySlides = 1, callback) {
// early return when there is nothing to slide
                if (this.innerElements.length <= this.perPage) {
                    return;
                }

                const beforeChange = this.currentSlide;

                if (this.config.loop) {
                    const isNewIndexClone = this.currentSlide - howManySlides < 0;
                    if (isNewIndexClone) {
                        this.disableTransition();

                        const mirrorSlideIndex = this.currentSlide + this.innerElements.length;
                        const mirrorSlideIndexOffset = this.perPage;
                        const moveTo = mirrorSlideIndex + mirrorSlideIndexOffset;
                        const offset = (this.config.rtl ? 1 : -1) * moveTo * (this.selectorWidth / this.perPage);
                        const dragDistance = this.config.draggable ? this.drag.endX - this.drag.startX : 0;

                        this.sliderFrame.style[this.transformProperty] = `translate3d(${offset + dragDistance}px, 0, 0)`;
                        this.currentSlide = mirrorSlideIndex - howManySlides;
                    } else {
                        this.currentSlide = this.currentSlide - howManySlides;
                    }
                } else {
                    this.currentSlide = Math.max(this.currentSlide - howManySlides, 0);
                }

                if (beforeChange !== this.currentSlide) {
                    this.slideToCurrent(this.config.loop);
                    this.config.onChange.call(this);
                    if (callback) {
                        callback.call(this);
                    }
                }
            },

            /**
             * Go to next slide.
             * @param {number} [howManySlides=1] - How many items to slide forward.
             * @param {function} callback - Optional callback function.
             */
            next(howManySlides = 1, callback) {
                // early return when there is nothing to slide
                if (this.innerElements.length <= this.perPage) {
                    return;
                }

                const beforeChange = this.currentSlide;

                if (this.config.loop) {
                    const isNewIndexClone = this.currentSlide + howManySlides > this.innerElements.length - this.perPage;
                    if (isNewIndexClone) {
                        this.disableTransition();

                        const mirrorSlideIndex = this.currentSlide - this.innerElements.length;
                        const mirrorSlideIndexOffset = this.perPage;
                        const moveTo = mirrorSlideIndex + mirrorSlideIndexOffset;
                        const offset = (this.config.rtl ? 1 : -1) * moveTo * (this.selectorWidth / this.perPage);
                        const dragDistance = this.config.draggable ? this.drag.endX - this.drag.startX : 0;

                        this.sliderFrame.style[this.transformProperty] = `translate3d(${offset + dragDistance}px, 0, 0)`;
                        this.currentSlide = mirrorSlideIndex + howManySlides;
                    } else {
                        this.currentSlide = this.currentSlide + howManySlides;
                    }
                } else {
                    this.currentSlide = Math.min(this.currentSlide + howManySlides, this.innerElements.length - this.perPage);
                }
                if (beforeChange !== this.currentSlide) {
                    this.slideToCurrent(this.config.loop);
                    this.config.onChange.call(this);
                    if (callback) {
                        callback.call(this);
                    }
                }
            },

            play(time = 6000) {
                this.time = time;
                this.playing = true;
                this.$options.play_timer = setTimeout(() => {
                    this.siema.next();
                }, time);
                this.$emit('update:playing', true)
            },

            stop() {
                this.playing = false;
                clearTimeout(this.$options.play_timer);
                this.$emit('update:playing', false);
            },
            /**
             * Disable transition on sliderFrame.
             */
            disableTransition() {
                this.sliderFrame.style.webkitTransition = `all 0ms ${this.config.easing}`;
                this.sliderFrame.style.transition = `all 0ms ${this.config.easing}`;
            },


            /**
             * Enable transition on sliderFrame.
             */
            enableTransition() {
                this.sliderFrame.style.webkitTransition = `all ${this.config.duration}ms ${this.config.easing}`;
                this.sliderFrame.style.transition = `all ${this.config.duration}ms ${this.config.easing}`;
            },


            /**
             * Go to slide with particular index
             * @param {number} index - Item index to slide to.
             * @param {function} callback - Optional callback function.
             */
            goTo(index, callback) {
                if (this.innerElements.length <= this.perPage) {
                    return;
                }
                const beforeChange = this.currentSlide;
                this.currentSlide = this.config.loop ?
                    index % this.innerElements.length :
                    Math.min(Math.max(index, 0), this.innerElements.length - this.perPage);
                if (beforeChange !== this.currentSlide) {
                    this.slideToCurrent();
                    this.config.onChange.call(this);
                    if (callback) {
                        callback.call(this);
                    }
                }
            },


            /**
             * Moves sliders frame to position of currently active slide
             */
            slideToCurrent(enableTransition) {
                const currentSlide = this.config.loop ? this.currentSlide + this.perPage : this.currentSlide;
                const offset = (this.config.rtl ? 1 : -1) * currentSlide * (this.selectorWidth / this.perPage);

                if (enableTransition) {
// This one is tricky, I know but this is a perfect explanation:
// https://youtu.be/cCOL7MC4Pl0
                    requestAnimationFrame(() => {
                        requestAnimationFrame(() => {
                            this.enableTransition();
                            this.sliderFrame.style[this.transformProperty] = `translate3d(${offset}px, 0, 0)`;
                        });
                    });
                } else {
                    this.sliderFrame.style[this.transformProperty] = `translate3d(${offset}px, 0, 0)`;
                }
            },


            /**
             * Recalculate drag /swipe event and reposition the frame of a slider
             */
            updateAfterDrag() {
                const movement = (this.config.rtl ? -1 : 1) * (this.drag.endX - this.drag.startX);
                const movementDistance = Math.abs(movement);
                const howManySliderToSlide = this.config.multipleDrag ? Math.ceil(movementDistance / (this.selectorWidth / this.perPage)) : 1;

                const slideToNegativeClone = movement > 0 && this.currentSlide - howManySliderToSlide < 0;
                const slideToPositiveClone = movement < 0 && this.currentSlide + howManySliderToSlide > this.innerElements.length - this.perPage;

                if (movement > 0 && movementDistance > this.config.threshold && this.innerElements.length > this.perPage) {
                    this.prev(howManySliderToSlide);
                } else if (movement < 0 && movementDistance > this.config.threshold && this.innerElements.length > this.perPage) {
                    this.next(howManySliderToSlide);
                }
                this.slideToCurrent(slideToNegativeClone || slideToPositiveClone);
            },


            /**
             * When window resizes, resize slider components as well
             */
            resizeHandler() {
// update perPage number dependable of user value
                this.resolveSlidesNumber();

// relcalculate currentSlide
// prevent hiding items when browser width increases
                if (this.currentSlide + this.perPage > this.innerElements.length) {
                    this.currentSlide = this.innerElements.length <= this.perPage ? 0 : this.innerElements.length - this.perPage;
                }

                this.selectorWidth = this.selector.offsetWidth;

                this.buildSliderFrame();
            },


            /**
             * Clear drag after touchend and mouseup event
             */
            clearDrag() {
                this.drag = {
                    startX: 0,
                    endX: 0,
                    startY: 0,
                    letItGo: null,
                    preventClick: this.drag.preventClick
                };
            },


            /**
             * touchstart event handler
             */
            touchstartHandler(e) {
// Prevent dragging / swiping on inputs, selects and textareas
                const ignoreSiema = ['TEXTAREA', 'OPTION', 'INPUT', 'SELECT'].indexOf(e.target.nodeName) !== -1;
                if (ignoreSiema) {
                    return;
                }

                e.stopPropagation();
                this.pointerDown = true;
                this.drag.startX = e.touches[0].pageX;
                this.drag.startY = e.touches[0].pageY;
            },


            /**
             * touchend event handler
             */
            touchendHandler(e) {
                e.stopPropagation();
                this.pointerDown = false;
                this.enableTransition();
                if (this.drag.endX) {
                    this.updateAfterDrag();
                }
                this.clearDrag();
            },


            /**
             * touchmove event handler
             */
            touchmoveHandler(e) {
                e.stopPropagation();

                if (this.drag.letItGo === null) {
                    this.drag.letItGo = Math.abs(this.drag.startY - e.touches[0].pageY) < Math.abs(this.drag.startX - e.touches[0].pageX);
                }

                if (this.pointerDown && this.drag.letItGo) {
                    e.preventDefault();
                    this.drag.endX = e.touches[0].pageX;
                    this.sliderFrame.style.webkitTransition = `all 0ms ${this.config.easing}`;
                    this.sliderFrame.style.transition = `all 0ms ${this.config.easing}`;

                    const currentSlide = this.config.loop ? this.currentSlide + this.perPage : this.currentSlide;
                    const currentOffset = currentSlide * (this.selectorWidth / this.perPage);
                    const dragOffset = (this.drag.endX - this.drag.startX);
                    const offset = this.config.rtl ? currentOffset + dragOffset : currentOffset - dragOffset;
                    this.sliderFrame.style[this.transformProperty] = `translate3d(${(this.config.rtl ? 1 : -1) * offset}px, 0, 0)`;
                }
            },


            /**
             * mousedown event handler
             */
            mousedownHandler(e) {
// Prevent dragging / swiping on inputs, selects and textareas
                const ignoreSiema = ['TEXTAREA', 'OPTION', 'INPUT', 'SELECT'].indexOf(e.target.nodeName) !== -1;
                if (ignoreSiema) {
                    return;
                }

                e.preventDefault();
                e.stopPropagation();
                this.pointerDown = true;
                this.drag.startX = e.pageX;
            },


            /**
             * mouseup event handler
             */
            mouseupHandler(e) {
                e.stopPropagation();
                this.pointerDown = false;
                this.selector.style.cursor = '-webkit-grab';
                this.enableTransition();
                if (this.drag.endX) {
                    this.updateAfterDrag();
                }
                this.clearDrag();
            },


            /**
             * mousemove event handler
             */
            mousemoveHandler(e) {
                e.preventDefault();
                if (this.pointerDown) {
// if dragged element is a link
// mark preventClick prop as a true
// to detemine about browser redirection later on
                    if (e.target.nodeName === 'A') {
                        this.drag.preventClick = true;
                    }

                    this.drag.endX = e.pageX;
                    this.selector.style.cursor = '-webkit-grabbing';
                    this.sliderFrame.style.webkitTransition = `all 0ms ${this.config.easing}`;
                    this.sliderFrame.style.transition = `all 0ms ${this.config.easing}`;

                    const currentSlide = this.config.loop ? this.currentSlide + this.perPage : this.currentSlide;
                    const currentOffset = currentSlide * (this.selectorWidth / this.perPage);
                    const dragOffset = (this.drag.endX - this.drag.startX);
                    const offset = this.config.rtl ? currentOffset + dragOffset : currentOffset - dragOffset;
                    this.sliderFrame.style[this.transformProperty] = `translate3d(${(this.config.rtl ? 1 : -1) * offset}px, 0, 0)`;
                }
            },


            /**
             * mouseleave event handler
             */
            mouseleaveHandler(e) {
                if (this.pointerDown) {
                    this.pointerDown = false;
                    this.selector.style.cursor = '-webkit-grab';
                    this.drag.endX = e.pageX;
                    this.drag.preventClick = false;
                    this.enableTransition();
                    this.updateAfterDrag();
                    this.clearDrag();
                }
            },


            /**
             * click event handler
             */
            clickHandler(e) {
// if the dragged element is a link
// prevent browsers from folowing the link
                if (this.drag.preventClick) {
                    e.preventDefault();
                }
                this.drag.preventClick = false;
            },


            /**
             * Remove item from carousel.
             * @param {number} index - Item index to remove.
             * @param {function} callback - Optional callback to call after remove.
             */
            remove(index, callback) {
                if (index < 0 || index >= this.innerElements.length) {
                    throw new Error('Item to remove doesn\'t exist 😭');
                }

                // Shift sliderFrame back by one item when:
                // 1. Item with lower index than currenSlide is removed.
                // 2. Last item is removed.
                const lowerIndex = index < this.currentSlide;
                const lastItem = this.currentSlide + this.perPage - 1 === index;

                if (lowerIndex || lastItem) {
                    this.currentSlide--;
                }

                this.innerElements.splice(index, 1);

                this.buildSliderFrame();// build a frame and slide to a currentSlide

                if (callback) {
                    callback.call(this);
                }
            },


            /**
             * Insert item to carousel at particular index.
             * @param {HTMLElement} item - Item to insert.
             * @param {number} index - Index of new new item insertion.
             * @param {function} callback - Optional callback to call after insert.
             */
            insert(item, index, callback) {
                if (index < 0 || index > this.innerElements.length + 1) {
                    throw new Error('Unable to inset it at this index 😭');
                }
                if (this.innerElements.indexOf(item) !== -1) {
                    throw new Error('The same item in a carousel? Really? Nope 😭');
                }

                const shouldItShift = index <= this.currentSlide > 0 && this.innerElements.length; // Avoid shifting content
                this.currentSlide = shouldItShift ? this.currentSlide + 1 : this.currentSlide;

                this.innerElements.splice(index, 0, item);

                this.buildSliderFrame();// build a frame and slide to a currentSlide

                if (callback) {
                    callback.call(this);
                }
            },

            /**
             * Prepernd item to carousel.
             * @param {HTMLElement} item - Item to prepend.
             * @param {function} callback - Optional callback to call after prepend.
             */
            prepend(item, callback) {
                this.insert(item, 0);
                if (callback) {
                    callback.call(this);
                }
            },


            /**
             * Append item to carousel.
             * @param {HTMLElement} item - Item to append.
             * @param {function} callback - Optional callback to call after append.
             */
            append(item, callback) {
                this.insert(item, this.innerElements.length + 1);
                if (callback) {
                    callback.call(this);
                }
            },


            /**
             * Removes listeners and optionally restores to initial markup
             * @param {boolean} restoreMarkup - Determinants about restoring an initial markup.
             * @param {function} callback - Optional callback function.
             */
            destroy(restoreMarkup = false, callback) {
                this.detachEvents();

                this.selector.style.cursor = 'auto';

                if (restoreMarkup) {
                    const slides = document.createDocumentFragment();
                    for (let i = 0; i < this.innerElements.length; i++) {
                        slides.appendChild(this.innerElements[i]);
                    }
                    this.selector.innerHTML = '';
                    this.selector.appendChild(slides);
                    this.selector.removeAttribute('style');
                }

                if (callback) {
                    callback.call(this);
                }
            },

        }
    }
</script>
