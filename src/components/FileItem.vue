<template>
    <li
        class="ww-file-item"
        :class="{ 'ww-file-item--disabled': isDisabled }"
        :style="fileItemStyles"
        role="listitem"
        :aria-label="`File: ${file.name}, Size: ${formattedSize}`"
    >
        <div
            v-if="status && status.uploadProgress !== undefined"
            class="ww-file-item__progress"
            :style="{
                width: `${Math.min(100, Math.round(status.uploadProgress))}%`,
                backgroundColor: content.progressBarColor || '#EEEEEE',
            }"
        ></div>
        <div class="ww-file-item__info">
            <div class="ww-file-item__name" :style="fileNameStyles">
                <input
                    v-if="isEditing && editableFilenames"
                    ref="filenameInput"
                    v-model="editedFilename"
                    type="text"
                    class="ww-file-item__name-input"
                    :style="fileNameInputStyles"
                    @blur="saveFilename"
                    @keydown.enter="saveFilename"
                    @keydown.esc="cancelEdit"
                />
                <span
                    v-else
                    :class="{ 'ww-file-item__name-editable': editableFilenames && !isReadonly && !isDisabled }"
                    @click="startEdit"
                >
                    {{ displayFilename }}
                    <span
                        v-if="editableFilenames && !isReadonly && !isDisabled"
                        class="ww-file-item__edit-icon"
                        v-html="editIcon"
                    ></span>
                </span>
            </div>
            <div class="ww-file-item__details" :style="fileDetailsStyles" v-if="showFileInfo">
                <span>{{ formattedSize }}</span>
                <span v-if="status && status.uploadProgress !== undefined">
                    • {{ `${Math.round(status.uploadProgress)}%` }}
                </span>
            </div>
        </div>
        <div class="ww-file-item__actions">
            <button
                v-if="!isReadonly"
                type="button"
                class="ww-file-item__btn ww-file-item__btn--remove"
                :disabled="isDisabled || isReadonly"
                @click="$emit('remove')"
                :style="actionButtonStyles"
                aria-label="Remove file"
            >
                <div class="icon" v-html="removeIcon"></div>
            </button>
        </div>
    </li>
</template>

<script>
import { computed, inject, onMounted, ref, nextTick } from 'vue';

export default {
    props: {
        file: {
            type: Object,
            required: true,
        },
        status: {
            type: Object,
            required: true,
        },
        index: {
            type: Number,
            required: true,
        },
        isReadonly: {
            type: Boolean,
            default: false,
        },
        isDisabled: {
            type: Boolean,
            default: false,
        },
    },
    emits: ['remove', 'rename'],
    setup(props, { emit }) {
        const fileUpload = inject('_wwFileUpload', {
            files: computed(() => []),
            content: computed(() => ({})),
            isDisabled: computed(() => false),
            isReadonly: computed(() => false),
            isSingleMode: computed(() => true),
            acceptedTypes: computed(() => ''),
        });

        const filesCount = computed(() => fileUpload.files.value.length);
        const content = computed(() => fileUpload.content?.value || {});
        const showFileInfo = computed(() => content.value?.showFileInfo);
        const editableFilenames = computed(() => content.value?.editableFilenames || false);

        // Filename editing state
        const isEditing = ref(false);
        const editedFilename = ref(props.file?.name || '');
        const filenameInput = ref(null);
        const displayFilename = computed(() => props.file?.name || '');

        const fileItemStyles = computed(() => ({
            backgroundColor: content.value?.fileItemBackground || '#fff',
            borderColor: content.value?.fileItemBorderColor || '#eee',
            borderRadius: content.value?.fileItemBorderRadius || '6px',
            padding: content.value?.fileItemPadding || '12px',
            margin: content.value?.fileItemMargin || '0 0 8px 0',
            boxShadow: content.value?.fileItemShadow || '0 2px 4px rgba(0, 0, 0, 0.05)',
            position: 'relative',
            overflow: 'hidden',
        }));

        const fileNameStyles = computed(() => ({
            fontFamily: content.value?.fileNameFontFamily || 'inherit',
            fontSize: content.value?.fileNameFontSize || '14px',
            fontWeight: content.value?.fileNameFontWeight || 500,
            color: content.value?.fileNameColor || 'inherit',
        }));

        const fileDetailsStyles = computed(() => ({
            fontFamily: content.value?.fileDetailsFontFamily || 'inherit',
            fontSize: content.value?.fileDetailsFontSize || '12px',
            fontWeight: content.value?.fileDetailsFontWeight || 'normal',
            color: content.value?.fileDetailsColor || '#888',
        }));

        const actionButtonStyles = computed(() => ({
            width: content.value?.actionButtonSize || '28px',
            height: content.value?.actionButtonSize || '28px',
            backgroundColor: content.value?.actionButtonBackground || '#fff',
            color: content.value?.actionButtonColor || '#666',
            borderRadius: content.value?.actionButtonBorderRadius || '4px',
            margin: content.value?.actionButtonMargin || '0 0 0 4px',
        }));

        const formattedSize = computed(() => {
            const fileSizeInBytes = props.file.size || 0;

            if (fileSizeInBytes === 0) return '0 B';

            const sizes = ['B', 'KB', 'MB', 'GB', 'TB'];
            const i = Math.floor(Math.log(fileSizeInBytes) / Math.log(1024));
            return `${(fileSizeInBytes / Math.pow(1024, i)).toFixed(2)} ${sizes[i]}`;
        });

        const { getIcon } = wwLib.useIcons();
        const removeIcon = ref(null);
        const editIcon = ref(null);

        onMounted(async () => {
            try {
                removeIcon.value = await getIcon('lucide/trash');
            } catch (e) {
                removeIcon.value = null;
            }

            try {
                editIcon.value = await getIcon('lucide/pencil');
            } catch (e) {
                // Fallback to inline SVG if icon fetch fails
                editIcon.value = `<svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/><path d="m15 5 4 4"/></svg>`;
            }
        });

        // Get file extension helper
        const getFileExtension = filename => {
            const lastDotIndex = filename?.lastIndexOf('.');
            if (lastDotIndex > 0) {
                return filename.substring(lastDotIndex);
            }
            return '';
        };

        // Filename editing methods
        const startEdit = () => {
            if (!editableFilenames.value || props.isReadonly || props.isDisabled) return;

            isEditing.value = true;
            editedFilename.value = props.file?.name || '';

            nextTick(() => {
                if (filenameInput.value) {
                    filenameInput.value.focus();
                    // Select filename without extension
                    const lastDotIndex = editedFilename.value.lastIndexOf('.');
                    if (lastDotIndex > 0) {
                        filenameInput.value.setSelectionRange(0, lastDotIndex);
                    } else {
                        filenameInput.value.select();
                    }
                }
            });
        };

        const saveFilename = () => {
            if (!isEditing.value) return;

            let newFilename = editedFilename.value?.trim();
            const oldFilename = props.file?.name || '';
            const extension = getFileExtension(oldFilename);

            /* wwEditor:start */
            console.log('[FileItem] saveFilename called', {
                newFilename,
                oldFilename,
                changed: newFilename !== oldFilename,
                isEmpty: !newFilename,
            });
            /* wwEditor:end */

            if (newFilename && newFilename !== oldFilename) {
                // Ensure the new filename has the extension
                if (!newFilename.endsWith(extension)) {
                    newFilename = newFilename + extension;
                }

                /* wwEditor:start */
                console.log('[FileItem] Emitting rename event', {
                    index: props.index,
                    oldName: oldFilename,
                    newName: newFilename,
                });
                /* wwEditor:end */
                emit('rename', {
                    index: props.index,
                    oldName: oldFilename,
                    newName: newFilename,
                });
            }

            isEditing.value = false;
        };

        const cancelEdit = () => {
            editedFilename.value = props.file?.name || '';
            isEditing.value = false;
        };

        const fileNameInputStyles = computed(() => ({
            fontFamily: content.value?.fileNameFontFamily || 'inherit',
            fontSize: content.value?.fileNameFontSize || '14px',
            fontWeight: content.value?.fileNameFontWeight || 500,
            color: content.value?.fileNameColor || 'inherit',
            backgroundColor: content.value?.fileItemBackground || '#fff',
            border: `1px solid ${content.value?.fileItemBorderColor || '#eee'}`,
            borderRadius: '4px',
            padding: '2px 4px',
            width: '100%',
            outline: 'none',
        }));

        return {
            formattedSize,
            filesCount,
            fileItemStyles,
            fileNameStyles,
            fileDetailsStyles,
            actionButtonStyles,
            content,
            showFileInfo,
            removeIcon,
            editIcon,
            editableFilenames,
            isEditing,
            editedFilename,
            filenameInput,
            displayFilename,
            startEdit,
            saveFilename,
            cancelEdit,
            fileNameInputStyles,
        };
    },
};
</script>

<style lang="scss" scoped>
.ww-file-item {
    display: flex;
    align-items: center;
    padding: v-bind('content?.fileItemPadding || "12px"');
    border: 1px solid v-bind('content?.fileItemBorderColor || "#eee"');
    border-radius: v-bind('content?.fileItemBorderRadius || "6px"');
    margin-bottom: v-bind('(content?.fileItemMargin || "0 0 8px 0").split(" ")[2] || "8px"');
    background-color: v-bind('content?.fileItemBackground || "#fff"');
    transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
    transform-origin: center;
    position: relative;
    z-index: 1;
    backface-visibility: hidden;
    will-change: transform, opacity;
    overflow: hidden;

    &__progress {
        position: absolute;
        top: 0;
        left: 0;
        height: 100%;
        z-index: 0;
        transition: width 1.2s ease;
        opacity: 0.2;
    }

    &:hover {
        border-color: v-bind('content?.fileItemHoverBorderColor || content?.fileItemBorderColor || "#ddd"');
        background-color: v-bind('content?.fileItemHoverBackground || content?.fileItemBackground || "#fff"');
        box-shadow: v-bind(
            'content?.fileItemHoverShadow || content?.fileItemShadow || "0 2px 4px rgba(0, 0, 0, 0.05)"'
        );
    }

    &--disabled {
        opacity: 0.6;
        pointer-events: none;
    }

    &__info {
        flex: 1;
        min-width: 0;
        position: relative;
        z-index: 1;
    }

    &__name {
        font-size: v-bind('content?.fileNameFontSize || "14px"');
        font-weight: v-bind('content?.fileNameFontWeight || 500');
        margin-bottom: 4px;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
        color: v-bind('content?.fileNameColor || "inherit"');

        &-editable {
            cursor: pointer;
            padding: 2px 6px 2px 4px;
            border-radius: 4px;
            transition: all 0.2s ease;
            display: inline-flex;
            align-items: center;
            gap: 6px;
            border: 1px solid transparent;

            &:hover {
                background-color: rgba(0, 0, 0, 0.04);
                border-color: v-bind('content?.progressBarColor || "rgba(76, 175, 80, 0.3)"');
                color: v-bind('content?.progressBarColor || "#4CAF50"');

                .ww-file-item__edit-icon {
                    opacity: 1;
                }
            }
        }

        &-input {
            font-family: inherit;
            font-size: inherit;
            font-weight: inherit;
            color: inherit;
            width: 100%;
            min-width: 0;
            padding: 2px 4px;
            border: 1px solid;
            border-radius: 4px;
            outline: none;
            transition: border-color 0.2s ease;

            &:focus {
                border-color: v-bind('content?.progressBarColor || "#4CAF50"');
                box-shadow: 0 0 0 2px
                    v-bind('content?.progressBarColor ? content.progressBarColor + "33" : "rgba(76, 175, 80, 0.2)"');
            }
        }
    }

    &__edit-icon {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        opacity: 0.5;
        transition: opacity 0.2s ease;
        flex-shrink: 0;
        width: 14px;
        height: 14px;
        margin-left: 2px;

        :deep(svg) {
            width: 100%;
            height: 100%;
            stroke: currentColor;
        }
    }

    &__details {
        font-size: v-bind('content?.fileDetailsFontSize || "12px"');
        color: v-bind('content?.fileDetailsColor || "#888"');
    }

    &__actions {
        display: flex;
        align-items: center;
        margin-left: 12px;
        opacity: 0.7;
        transition: opacity 0.2s ease;
        position: relative;
        z-index: 1;
    }

    &:hover &__actions {
        opacity: 1;
    }

    &__btn {
        display: flex;
        align-items: center;
        justify-content: center;
        width: v-bind('content?.actionButtonSize || "28px"');
        height: v-bind('content?.actionButtonSize || "28px"');
        border-radius: v-bind('content?.actionButtonBorderRadius || "4px"');
        border: 1px solid v-bind('content?.actionButtonBorderColor || content?.fileItemBorderColor || "#eee"');
        background-color: v-bind('content?.actionButtonBackground || "#fff"');
        color: v-bind('content?.actionButtonColor || "#666"');
        margin-left: v-bind('(content?.actionButtonMargin || "0 0 0 4px").split(" ")[3] || "4px"');
        cursor: pointer;
        transition: all 0.2s ease;

        &:hover:not(:disabled) {
            border-color: v-bind(
                'content?.actionButtonHoverBorderColor || content?.fileItemHoverBorderColor || "#ddd"'
            );
            background-color: v-bind('content?.actionButtonHoverBackground || "#f8f8f8"');
            transform: scale(1.05);
        }

        &:disabled {
            opacity: 0.5;
            cursor: not-allowed;
        }

        &--remove:hover:not(:disabled) {
            color: v-bind('content?.actionButtonRemoveHoverColor || content?.progressBarColor || "#999"');
            border-color: v-bind('content?.actionButtonRemoveHoverColor || content?.progressBarColor || "#999"');
        }

        .icon {
            width: 50%;
            height: 50%;
            display: flex;
            align-items: center;
            justify-content: center;

            :deep(svg) {
                width: 100% !important;
                height: 100% !important;
                display: block;
            }
        }
    }
}
</style>
