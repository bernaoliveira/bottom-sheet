<template>
    <transition name="fade">
        <div :class="$style.wrapper">
            <div
                :class="[$style.bg, transparentBg ? $style._transparent : '']"
                @click.stop="onClose"
            />
            <div
                ref="card"
                :style="{ top: y !== null ? `${ y }px` : ''}"
                :class="$style.card"
                :data-state="isMoving ? 'move' : 'open'"
            >
                <div
                    ref="pan"
                    v-touch:start="handleMoveStart"
                    v-touch:end="handleMoveEnd"
                    v-touch:moving="handleMove"
                    :class="$style.panArea"
                >
                    <div ref="bar" :class="$style.bar" />
                </div>
                <div ref="sheetContents" :class="$style.sheetContents">
                    <!-- Провайдим метод закрытия в слот -->
                    <!-- Метод будет доступен через <template #default="{ close }"> -->
                    <slot :close="onClose"></slot>
                </div>
            </div>
        </div>
    </transition>
</template>

<script>
import { lockBody, unlockBody } from '~/assets/js/utils/common-utils';

export default {
    name: 'BottomSheet',

    props: {
        transparentBg: {
            type: Boolean,
            default: false,
        },

        needLockBody: {
            type: Boolean,
            default: true,
        },
    },

    data() {
        return {
            startY: null,
            y: null,

            // Взаимодействие с DOM
            rect: null,
            offsetHeight: null,
            observer: null,

            isMoving: false,
        };
    },

    watch: {
        isMoving(value) {
            if (value) {
                this.$emit('move');
            }
        },

        // При изменении высоты контента внтури карточки меняем её положение
        offsetHeight(value, oldValue) {
            if (!this.$refs.card) {
                return;
            }

            this.rect = this.$refs.card.getBoundingClientRect();

            if (this.y < 0) {
                this.y = 0;
                return;
            }

            // Если высота контента меньше, чем высота экрана
            if (value < this.rect.height) {
                this.y = window.innerHeight - value - 20; // 20 - высота panArea
                return;
            }

            if (value && oldValue) {
                this.y -= value - oldValue;
            }
        },
    },

    mounted() {
        if (this.$refs.card) {
            // Подписываемся на изменение размеров карточки
            this.observer = new MutationObserver(() => {
                this.$nextTick(() => {
                    this.offsetHeight = this.$refs.sheetContents.scrollHeight;
                });
            });

            const config = { attributes: true, childList: true, subtree: true };

            this.observer.observe(this.$refs.card, config);

            // Получаем начальное положение карточки
            // Таймаут нужен, чтобы карточка появиться после анимации открытия модалки
            setTimeout(() => {
                if (this.$refs.card) {
                    this.rect = this.$refs.card.getBoundingClientRect();
                    this.y = this.rect.top;

                    // Если высота карточки больше, чем высота экрана, то устанавливаем отступ сверху - 0
                    if (this.rect.height > window.innerHeight) {
                        this.y = 0;
                    }
                }
            }, 600);
        }

        if (this.needLockBody) {
            lockBody();
        }
    },

    beforeDestroy() {
        this.observer.disconnect();

        if (this.needLockBody) {
            unlockBody();
        }
    },

    methods: {
        handleMoveStart(event) {
            this.isMoving = true;
            if (this.y === null) {
                this.startY = event.changedTouches[0].clientY - 16;
            } else {
                this.startY = this.y;
            }
        },

        handleMove(event) {
            // Получаем y координату тача и вычитаем 16 пикселей,
            // чтобы визуально карточка не опаздывала за пальцем
            this.y = event.changedTouches[0].clientY - 16;
        },

        handleMoveEnd() {
            this.isMoving = false;

            // Высота panArea - 20px
            // Если начальное положение в пределах panArea,
            // А новое значение меньше 0 (пользователь свайпнул вверх),
            // То возвращаем карточку в начальное положение
            if (this.startY < 20 - 16 && this.y < -16) {
                this.y = 0;
                return;
            }

            // Сравниваем текущее положение карточки с начальным
            // Если карточка выше чем была, то возвращаем её в начальное положение
            if (this.y < this.startY) {
                this.y = this.startY;
                return;
            }

            // Детектим свайп вниз + положение карточки относительно начальной точки
            if (this.y > this.startY) {
                this.$emit('close');
            }
        },

        onClose(closeAll = false) {
            if (this.rect?.top && this.rect?.height) {
                this.y = this.rect.top + this.rect.height;
            }

            if (closeAll === true) {
                this.$modal.close();
            }
            setTimeout(() => {
                this.$emit('close');
            }, 200);
        },
    },
};
</script>

<style lang="scss" module>
    .wrapper {
        position: fixed;
        top: 0;
        left: 0;
        z-index: 100;

        .bg {
            position: fixed;
            top: 0;
            right: 0;
            bottom: 0;
            left: 0;
            display: block;
            background: rgba(55, 58, 65, .87);
            transition: all .3s;

            &._transparent {
                background: transparent;
            }
        }
    }

    .card {
        position: fixed;
        bottom: 0;
        left: 0;
        width: 100%;
        height: auto;
        max-height: calc(var(--vh, 1vh) * 100);
        background-color: $base-0;
        color: $base-500;
        animation: slideUp .3s ease-out;

        &[data-state='open'] {
            transition: top .3s ease-out;
        }
    }

    .bar {
        width: 4rem;
        height: .4rem;
        margin: 0 auto;
        border-radius: 1rem;
        background: #e0e0e0;
        cursor: pointer;
    }

    .panArea {
        position: absolute;
        top: 0;
        right: 0;
        left: 0;
        display: flex;
        align-items: center;
        height: 2rem;

        .bar {
            position: relative;

            &:hover {
                cursor: grab;
            }

            &:active {
                cursor: grabbing;
            }
        }
    }

    .title {
        margin: 40px 20px 0;
    }

    .sheetContents {
        box-sizing: border-box;
        overflow-x: hidden;
        overflow-y: auto;
        display: flex;
        flex-direction: column;
        height: auto;
        max-height: calc(var(--vh, 1vh) * 100 - 2rem);
        margin-top: 2rem;

        &::-webkit-scrollbar {
            width: 0;
        }
    }

    @keyframes slideUp {
        from {
            transform: translateY(100%);
        }

        to {
            transform: translateY(0);
        }
    }
</style>
