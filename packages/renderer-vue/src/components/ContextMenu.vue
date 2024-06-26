<template>
    <transition name="slide-fade">
        <div v-show="modelValue" ref="el" class="baklava-context-menu" :class="classes" :style="styles">
            <template v-for="(item, index) in itemsWithHoverProperty">
                <div v-if="item.isDivider" :key="`d-${index}`" class="divider" />

                <div
                    v-else
                    :key="`i-${index}`"
                    class="item"
                    :class="{ 'submenu': !!item.submenu, '--disabled': !!item.disabled }"
                    @mouseenter="onMouseEnter($event, index)"
                    @mouseleave="onMouseLeave($event, index)"
                    @click.stop.prevent="onClick(item)"
                >
                    <div class="flex-fill">
                        {{ item.label }}
                    </div>
                    <div v-if="item.submenu" class="__submenu-icon" style="line-height: 1em">
                        <svg width="13" height="13" viewBox="-60 120 250 250">
                            <path
                                d="M160.875 279.5625 L70.875 369.5625 L70.875 189.5625 L160.875 279.5625 Z"
                                stroke="none"
                                fill="white"
                            />
                        </svg>
                    </div>
                    <context-menu
                        v-if="item.submenu"
                        :value="activeMenu === index"
                        :items="item.submenu"
                        :is-nested="true"
                        :is-flipped="{ x: flippedX, y: flippedY }"
                        :flippable="flippable"
                        @click="onChildClick"
                    />
                </div>
            </template>
        </div>
    </transition>
</template>

<script lang="ts">
import { computed, defineComponent, Ref, ref, watch } from "vue";
import { onClickOutside } from "@vueuse/core";

export interface IMenuItem {
    label?: string;
    value?: any;
    isDivider?: boolean;
    submenu?: IMenuItem[];
    disabled?: boolean | Readonly<Ref<boolean>>;
}

export default defineComponent({
    props: {
        modelValue: {
            type: Boolean,
            default: false,
        },
        items: {
            type: Array as () => IMenuItem[],
            required: true,
        },
        x: {
            type: Number,
            default: 0,
        },
        y: {
            type: Number,
            default: 0,
        },
        isNested: {
            type: Boolean,
            default: false,
        },
        isFlipped: {
            type: Object as () => { x: boolean; y: boolean },
            default: () => ({ x: false, y: false }),
        },
        flippable: {
            type: Boolean,
            default: false,
        },
    },
    emits: ["click", "update:modelValue"],
    setup(props, { emit }) {
        let activeMenuResetTimeout: number | null = null;

        const el = ref<HTMLElement | null>(null);
        const activeMenu = ref(-1);
        const height = ref(0);
        const rootIsFlipped = ref({ x: false, y: false });

        const flippedX = computed(() => props.flippable && (rootIsFlipped.value.x || props.isFlipped.x));
        const flippedY = computed(() => props.flippable && (rootIsFlipped.value.y || props.isFlipped.y));

        const styles = computed(() => {
            const s: any = {};
            if (!props.isNested) {
                s.top = (flippedY.value ? props.y - height.value : props.y) + "px";
                s.left = props.x + "px";
            }
            return s;
        });

        const classes = computed(() => {
            return {
                "--flipped-x": flippedX.value,
                "--flipped-y": flippedY.value,
                "--nested": props.isNested,
            };
        });

        const itemsWithHoverProperty = computed(() => props.items.map((i) => ({ ...i, hover: false })));

        watch([() => props.y, () => props.items], () => {
            height.value = props.items.length * 30;
            const parentWidth = el.value?.parentElement?.offsetWidth ?? 0;
            const parentHeight = el.value?.parentElement?.offsetHeight ?? 0;
            rootIsFlipped.value.x = !props.isNested && props.x > parentWidth * 0.75;
            rootIsFlipped.value.y = !props.isNested && props.y + height.value > parentHeight - 20;
        });

        onClickOutside(el, () => {
            if (props.modelValue) {
                emit("update:modelValue", false);
            }
        });

        const onClick = (item: IMenuItem) => {
            if (!item.submenu && item.value) {
                emit("click", item.value);
                emit("update:modelValue", false);
            }
        };

        const onChildClick = (value: string) => {
            emit("click", value);
            activeMenu.value = -1;
            if (!props.isNested) {
                emit("update:modelValue", false);
            }
        };

        const onMouseEnter = (event: MouseEvent, index: number) => {
            if (props.items[index].submenu) {
                activeMenu.value = index;
                if (activeMenuResetTimeout !== null) {
                    clearTimeout(activeMenuResetTimeout);
                    activeMenuResetTimeout = null;
                }
            }
        };

        const onMouseLeave = (event: MouseEvent, index: number) => {
            if (props.items[index].submenu) {
                activeMenuResetTimeout = window.setTimeout(() => {
                    activeMenu.value = -1;
                    activeMenuResetTimeout = null;
                }, 200);
            }
        };

        return {
            el,
            activeMenu,
            flippedX,
            flippedY,
            styles,
            classes,
            itemsWithHoverProperty,
            onClick,
            onChildClick,
            onClickOutside,
            onMouseEnter,
            onMouseLeave,
        };
    },
});
</script>
