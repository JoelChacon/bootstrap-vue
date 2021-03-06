<template>
    <div>
        <span ref="trigger"><slot></slot></span>

        <div tabindex="-1" class="popover fade" :class="[classState ? 'show' : '', popoverAlignment]" ref="popover" @focus="$emit('focus')"
             @blur="$emit('blur')" :style="popoverStyle">
            <div class="popover-arrow"></div>
            <h3 class="popover-title" v-if="title" v-html="title"></h3>
            <div class="popover-content">
                <div class="popover-content-wrapper">
                    <slot name="content"><span v-html="content"></span></slot>
                </div>
            </div>
        </div>

    </div>
</template>

<script>
    import Tether from 'tether';

    // Controls which events are mapped for each named trigger, and the expected popover behavior for each.
    const TRIGGER_LISTENERS = {
        click: {click: 'toggle'},
        hover: {mouseenter: 'show', mouseleave: 'hide'},
        focus: {focus: 'show', blur: 'hide'}
    };

    const PLACEMENT_PARAMS = {
        top: {
            attachment: 'bottom center',
            targetAttachment: 'top center'
        },
        bottom: {
            attachment: 'top center',
            targetAttachment: 'bottom center'
        },
        left:
        {
            attachment: 'middle right',
            targetAttachment: 'middle left'
        },
        right: {
            attachment: 'middle left',
            targetAttachment: 'middle right'
        }
    };

    const TETHER_CLASS_PREFIX = 'tether-';

    const TRANSITION_DURATION = 150;

    export default {
        props: {
            constraints: {
                type: Array,
                default() {
                    return [];
                }
            },
            content: {
                type: String,
                default: ''
            },
            debounce: {
                type: [Number],
                default: 300,
                validator(value) {
                    return value >= 0;
                }
            },
            delay: {
                type: [Number, Object],
                default: 0,
                validator(value) {
                    if (typeof value === 'number') {
                        return value >= 0;
                    } else if (value !== null && typeof value === 'object') {
                        return typeof value.show === 'number' &&
                            typeof value.hide === 'number' &&
                            value.show >= 0 &&
                            value.hide >= 0;
                    }

                    return false;
                }
            },
            offset: {
                type: String,
                default: '0 0',
                validator(value) {
                    // Regex test for a pair of units, either 0 exactly, px, or percentage
                    return /^((0\s?)|([+-]?[0-9]+(px|%)\s?)){2}$/.test(value);
                }
            },
            placement: {
                type: String,
                default: 'top',
                validator(value) {
                    return Object.keys(PLACEMENT_PARAMS).indexOf(value) !== -1;
                }
            },
            popoverStyle: {
                type: Object,
                default: null
            },
            show: {
                type: Boolean,
                default: null
            },
            targetOffset: {
                type: String,
                default: '0 0',
                validator(value) {
                    // Regex test for a pair of units, either 0 exactly, px, or percentage
                    return /^((0\s?)|([+-]?[0-9]+(px|%)\s?)){2}$/.test(value);
                }
            },
            title: {
                type: String,
                default: ''
            },
            triggers: {
                type: [Boolean, String, Array],
                default: () => ['click', 'focus'],
                validator(value) {
                    // Allow falsy value to disable all event triggers (equivalent to 'manual') in Bootstrap 4
                    if (value === false || value === '') {
                        return true;
                    } else if (typeof value === 'string') {
                        return Object.keys(TRIGGER_LISTENERS).indexOf(value) !== -1;
                    } else if (Array.isArray(value)) {
                        const keys = Object.keys(TRIGGER_LISTENERS);
                        value.forEach(item => {
                            if (keys.indexOf(item) === -1) {
                                return false;
                            }
                        });
                        return true;
                    }
                    return false;
                }
            }
        },

        data() {
            return {
                triggerState: this.show,
                classState: this.show,
                lastEvent: null
            };
        },

        computed: {
            /**
             * Arrange event trigger hooks as array for all input types.
             *
             * @return Array
             */
            normalizedTriggers() {
                if (this.triggers === false) {
                    return [];
                } else if (typeof this.triggers === 'string') {
                    return [this.triggers];
                }
                return this.triggers;
            },

            /**
             * Class property to be used for Popover rendering
             *
             * @return String
             */
            popoverAlignment() {
                return !this.placement || this.placement === `default` ? `popover-top` : `popover-${this.placement}`;
            },

            /**
             * Determine if the Popover should be shown.
             *
             * @return Boolean
             */
            showState() {
                return this.show !== false && (this.triggerState || this.show);
            }
        },

        watch: {
            /**
             * Refresh Tether display properties
             */
            constraints() {
                this.setOptions();
            },

            /**
             * Refresh Popover event triggers
             * @param {Array} newTriggers
             * @param {Array} oldTriggers
             */
            normalizedTriggers(newTriggers, oldTriggers) {
                this.updateListeners(newTriggers, oldTriggers);
            },

            /**
             * Refresh Tether display properties
             */
            offset() {
                this.setOptions();
            },

            /**
             * Refresh Tether display properties
             */
            placement() {
                this.setOptions();
            },

            /**
             * Affect 'show' state in response to status change
             * @param  {Boolean} val
             */
            showState(val) {
                const delay = this.getDelay(val);

                clearTimeout(this._timeout);

                if (delay) {
                    this._timeout = setTimeout(() => this.togglePopover(val), delay);
                } else {
                    this.togglePopover(val);
                }
            }
        },

        methods: {
            /**
             * Add all event hooks for the given trigger
             * @param {String} trigger
             */
            addListener(trigger) {
                // eslint-disable-next-line guard-for-in
                for (const item in TRIGGER_LISTENERS[trigger]) {
                    this._trigger.addEventListener(item, e => this.eventHandler(e));
                }
            },

            /**
             * Tidy removal of Tether object from the DOM
             */
            destroyTether() {
                if (this._tether && !this.showState) {
                    this._tether.destroy();
                    this._tether = null;

                    const regx = new RegExp('(^|[^-]\\b)(' + TETHER_CLASS_PREFIX + '\\S*)', 'g');
                    this._trigger.className = this._trigger.className.replace(regx, '');
                }
            },

            /**
             * Handle multiple event triggers
             * @param  {Object} e
             */
            eventHandler(e) {
                // If this event is right after a previous successful event, ignore it (debounce)
                if (this.normalizedTriggers.length > 1 && this.debounce > 0 && this.lastEvent !== null && e.timeStamp <= this.lastEvent + this.debounce) {
                    return;
                }

                // Look up the expected popover action for the event
                // eslint-disable-next-line guard-for-in
                for (const trigger in TRIGGER_LISTENERS) {
                    for (const event in TRIGGER_LISTENERS[trigger]) {
                        if (event === e.type) {
                            const action = TRIGGER_LISTENERS[trigger][event];

                            // If the expected event action is the opposite of the current state, allow it
                            if (action === 'toggle' || (this.triggerState && action === 'hide') || (!this.triggerState && action === 'show')) {
                                this.triggerState = !this.triggerState;
                                this.lastEvent = e.timeStamp;
                            }
                            return;
                        }
                    }
                }
            },

            /**
             * Get the currently applicable popover delay
             *
             * @returns Number
             */
            getDelay(state) {
                if (typeof this.delay === 'object') {
                    return state ? this.delay.show : this.delay.hide;
                }

                return this.delay;
            },

            /**
             * Tether construct params for each show event.
             *
             * @return Object
             */
            getTetherOptions() {
                return {
                    element: this._popover,
                    target: this._trigger,
                    constraints: this.constraints,
                    attachment: PLACEMENT_PARAMS[this.placement].attachment,
                    targetAttachment: PLACEMENT_PARAMS[this.placement].targetAttachment,
                    offset: this.offset,
                    targetOffset: this.targetOffset
                };
            },

            /**
             * Hide popover and fire event
             */
            hidePopover() {
                this.classState = false;
                clearTimeout(this._timeout);
                this._timeout = setTimeout(() => {
                    this._popover.style.display = 'none';
                    this.destroyTether();
                }, TRANSITION_DURATION);
            },

            /**
             * Refresh the Popover position in order to respond to changes
             */
            refreshPosition() {
                if (this._tether) {
                    this.$nextTick(() => {
                        this._tether.position();
                    });
                }
            },

            /**
             * Remove all event hooks for the given trigger
             * @param {String} trigger
             */
            removeListener(trigger) {
                // eslint-disable-next-line guard-for-in
                for (const item in TRIGGER_LISTENERS[trigger]) {
                    this._trigger.removeEventListener(item, e => this.eventHandler(e));
                }
            },

            /**
             * Update tether options
             */
            setOptions() {
                if (this._tether) {
                    this._tether.setOptions(this.getTetherOptions());
                }
            },

            /**
             * Display popover and fire event
             */
            showPopover() {
                clearTimeout(this._timeout);

                if (!this._tether) {
                    this._tether = new Tether(this.getTetherOptions());
                }
                this._popover.style.display = 'block';

                // Make sure the popup is rendered in the correct location
                this.refreshPosition();

                this.$nextTick(() => {
                    this.classState = true;
                });
            },

            /**
             * Handle Popover show or hide instruction
             */
            togglePopover(newShowState) {
                this.$emit('showChange', newShowState);
                if (newShowState) {
                    this.showPopover();
                    this.$root.$emit('shown::popover');
                } else {
                    this.hidePopover();
                    this.$root.$emit('hidden::popover');
                }
            },

            /**
             * Study the 'triggers' component property and apply all selected triggers
             * @param {Array} triggers
             * @param {Array} appliedTriggers
             */
            updateListeners(triggers, appliedTriggers = []) {
                const newTriggers = [];
                const removeTriggers = [];

                // Look for new events not yet mapped (all of them on first load)
                triggers.forEach(item => {
                    if (appliedTriggers.indexOf(item) === -1) {
                        newTriggers.push(item);
                    }
                });

                // Disable any removed event triggers
                appliedTriggers.forEach(item => {
                    if (triggers.indexOf(item) === -1) {
                        removeTriggers.push(item);
                    }
                });

                // Apply trigger mapping changes
                newTriggers.forEach(item => this.addListener(item));
                removeTriggers.forEach(item => this.removeListener(item));
            }
        },

        created() {
            this.$root.$on('hide::popover', () => {
                this.triggerState = false;
            });
        },

        mounted() {
            // Configure tether
            this._trigger = this.$refs.trigger.children[0];
            this._popover = this.$refs.popover;
            this._timeout = 0;

            // Add listeners for specified triggers and complementary click event
            this.updateListeners(this.normalizedTriggers);

            // Display popover if prop is set on load
            if (this.showState) {
                this.showPopover();
            }
        },

        updated() {
            this.refreshPosition();
        },

        beforeDestroy() {
            this.normalizedTriggers.forEach(item => this.removeListener(item));
            clearTimeout(this._timeout);
            this.destroyTether();
        }
    };

</script>
