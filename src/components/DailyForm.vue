<script setup>
import { ref, onMounted, onUnmounted, defineProps, watch, computed } from "vue";

const props = defineProps(["selectedDate"]);

const sections = [
    {
        id: "completed",
        label: "Việc đã hoàn thành",
        placeholder: "Những công việc bạn đã hoàn thành trong hôm nay...",
    },
    {
        id: "notes",
        label: "Ghi chú",
        placeholder: "",
    },
    {
        id: "next",
        label: "Kế hoạch tiếp theo",
        placeholder: "Dự định công việc cho ngày mai...",
    },
    {
        id: "blockers",
        label: "Khó khăn (Blockers)",
        placeholder: "Các vấn đề đang cản trở tiến độ của bạn...",
    },
];

// Trạng thái dữ liệu báo cáo
const reportData = ref({
    completed: "",
    notes: [],
    next: "",
    blockers: "",
});

const addNote = () => {
    reportData.value.notes.push({
        id: Date.now(),
        title: "Ghi chú mới",
        content: "",
    });
};

const showDeleteNoteModal = ref(false);
const noteToDeleteIndex = ref(null);

const removeNote = (index) => {
    noteToDeleteIndex.value = index;
    showDeleteNoteModal.value = true;
};

const executeRemoveNote = () => {
    if (noteToDeleteIndex.value !== null) {
        reportData.value.notes.splice(noteToDeleteIndex.value, 1);
        showDeleteNoteModal.value = false;
        noteToDeleteIndex.value = null;
        showToast("Đã xóa ghi chú con!");
    }
};

// Trạng thái lưu
const saveStatus = ref("saved"); // 'saved', 'saving', 'unsaved'

// Trạng thái phóng to (Focus Mode)
const zoomedSection = ref(null); // ID của section đang được phóng to

// Trạng thái thu gọn (Collapse) - Mặc định thu gọn phần next và blockers
const collapsedSections = ref(["next", "blockers"]);

let isDeleting = false;
const showConfirmModal = ref(false);

const confirmDelete = () => {
    showConfirmModal.value = true;
};

const executeDelete = () => {
    showConfirmModal.value = false;
    const dateKey = props.selectedDate;

    localStorage.removeItem(`dr_data_${dateKey}`);

    let history = JSON.parse(localStorage.getItem("dr_history") || "[]");
    history = history.filter((d) => d !== dateKey);
    localStorage.setItem("dr_history", JSON.stringify(history));

    window.dispatchEvent(new CustomEvent("report-saved"));

    isDeleting = true;
    reportData.value = { completed: "", notes: [], next: "", blockers: "" };
    saveStatus.value = "saved";

    setTimeout(() => {
        isDeleting = false;
    }, 100);

    showToast("Đã xóa báo cáo!");

    const today = new Date();
    const todayKey = `${today.getFullYear()}-${String(today.getMonth() + 1).padStart(2, "0")}-${String(today.getDate()).padStart(2, "0")}`;
    if (dateKey !== todayKey) {
        window.dispatchEvent(
            new CustomEvent("change-date", { detail: todayKey }),
        );
    }
};

// Thông báo (Dispatch global toast)
const showToast = (message, type = "success") => {
    window.dispatchEvent(
        new CustomEvent("show-toast", { detail: { message, type } }),
    );
};

// Xử lý hiển thị ngày tháng
const displayDate = computed(() => {
    if (!props.selectedDate) return "";
    const [year, month, day] = props.selectedDate.split("-");
    const date = new Date(year, month - 1, day);
    return date.toLocaleDateString("vi-VN", {
        day: "2-digit",
        month: "long",
        year: "numeric",
    });
});

const isToday = computed(() => {
    const today = new Date();
    const todayKey = `${today.getFullYear()}-${String(today.getMonth() + 1).padStart(2, "0")}-${String(today.getDate()).padStart(2, "0")}`;
    return props.selectedDate === todayKey;
});

// Refs cho textareas
const textareaRefs = {};
const setRef = (id, el) => {
    if (el) textareaRefs[id] = el;
};

// Tải dữ liệu
const loadReport = (dateKey) => {
    if (!dateKey) return;
    reportData.value = { completed: "", notes: [], next: "", blockers: "" };
    const saved = localStorage.getItem(`dr_data_${dateKey}`);
    if (saved) {
        try {
            reportData.value = JSON.parse(saved);
            if (!reportData.value.notes) {
                reportData.value.notes = [];
            }
            saveStatus.value = "saved";
        } catch (e) {
            console.error("Lỗi khi tải dữ liệu:", e);
        }
    }
};

// Lưu dữ liệu
const performSave = (isAuto = false) => {
    const dateKey = props.selectedDate;
    localStorage.setItem(
        `dr_data_${dateKey}`,
        JSON.stringify(reportData.value),
    );

    const history = JSON.parse(localStorage.getItem("dr_history") || "[]");
    if (!history.includes(dateKey)) {
        history.push(dateKey);
        localStorage.setItem(
            "dr_history",
            JSON.stringify(history.sort().reverse()),
        );
        window.dispatchEvent(new CustomEvent("report-saved"));
    }

    saveStatus.value = "saved";
    if (!isAuto) {
        showToast("Đã lưu báo cáo thành công!");
    }
};

const saveReport = () => {
    saveStatus.value = "saving";
    performSave(false);
};

let saveTimeout = null;
const triggerAutoSave = () => {
    saveStatus.value = "unsaved";
    if (saveTimeout) clearTimeout(saveTimeout);
    saveTimeout = setTimeout(() => {
        saveStatus.value = "saving";
        performSave(true);
    }, 1000);
};

// Phóng to / Thu nhỏ section
let scrollPosition = 0;

const toggleZoom = (id) => {
    if (zoomedSection.value === id) {
        zoomedSection.value = null;
        document.body.style.overflow = "";
        document.body.style.minHeight = "";
        setTimeout(() => {
            window.scrollTo(0, scrollPosition);
        }, 0);
    } else {
        scrollPosition = window.scrollY;
        document.body.style.minHeight = `${document.documentElement.scrollHeight}px`;
        document.body.style.overflow = "hidden";
        zoomedSection.value = id;
    }
};

// Thu gọn / Mở rộng section
const toggleCollapse = (id) => {
    const index = collapsedSections.value.indexOf(id);
    if (index === -1) {
        collapsedSections.value.push(id);
    } else {
        collapsedSections.value.splice(index, 1);
    }
};

// Sao chép báo cáo
const copySection = (id, label) => {
    let content = "";
    if (id === "notes") {
        const notes = reportData.value.notes || [];
        if (notes.length === 0) {
            content = "(Trống)";
        } else {
            content = notes
                .map((note) => `[${note.title}]\n${note.content || ""}`)
                .join("\n\n");
        }
    } else {
        content = reportData.value[id] || "(Trống)";
    }

    navigator.clipboard
        .writeText(content)
        .then(() => {
            showToast(`Đã sao chép "${label}"!`);
        })
        .catch(() => {
            showToast("Không thể sao chép!", "error");
        });
};

const copySubNote = (index) => {
    const note = reportData.value.notes[index];
    if (!note) return;
    const content = `[${note.title}]\n${note.content || ""}`;
    navigator.clipboard
        .writeText(content)
        .then(() => {
            showToast(`Đã sao chép "${note.title}"!`);
        })
        .catch(() => {
            showToast("Không thể sao chép!", "error");
        });
};

// Xử lý thụt lề
const adjustIndent = (id, amount, noteIndex = null) => {
    const refId = noteIndex !== null ? `notes_${noteIndex}` : id;
    const el = textareaRefs[refId];
    if (!el) return;
    const start = el.selectionStart;
    const end = el.selectionEnd;
    const value =
        noteIndex !== null
            ? reportData.value.notes[noteIndex].content
            : reportData.value[id];

    const updateValue = (newValue) => {
        if (noteIndex !== null) {
            reportData.value.notes[noteIndex].content = newValue;
        } else {
            reportData.value[id] = newValue;
        }
    };

    const beforeStart = value.substring(0, start);
    const lastNewLine = beforeStart.lastIndexOf("\n");
    const lineStart = lastNewLine === -1 ? 0 : lastNewLine + 1;
    const afterLineStart = value.substring(lineStart);

    const levels = [
        { marker: "-", indent: "" },
        { marker: "+", indent: "  " },
        { marker: "*", indent: "    " },
        { marker: "**", indent: "      " },
    ];

    let currentLevel = -1;
    let currentMatch = null;
    for (let i = 0; i < levels.length; i++) {
        const escapedMarker = levels[i].marker.replace(
            /[.*+?^${}()|[\]\\]/g,
            "\\$&",
        );
        const regex = new RegExp(`^(${levels[i].indent}${escapedMarker}\\s+)`);
        const match = afterLineStart.match(regex);
        if (match) {
            currentLevel = i;
            currentMatch = match[1];
            break;
        }
    }

    if (currentLevel !== -1) {
        let nextLevel = currentLevel + (amount > 0 ? 1 : -1);
        if (nextLevel < 0) nextLevel = 0;
        if (nextLevel >= levels.length) nextLevel = levels.length - 1;
        if (nextLevel !== currentLevel) {
            const newPrefix = `${levels[nextLevel].indent}${levels[nextLevel].marker} `;
            const diff = newPrefix.length - currentMatch.length;
            updateValue(
                value.substring(0, lineStart) +
                    newPrefix +
                    afterLineStart.substring(currentMatch.length),
            );
            setTimeout(() => {
                el.selectionStart = start + diff;
                el.selectionEnd = end + diff;
                el.focus();
                triggerAutoSave();
            }, 0);
            return;
        }
    }

    if (amount > 0) {
        updateValue(
            value.substring(0, lineStart) + "  " + value.substring(lineStart),
        );
        setTimeout(() => {
            el.selectionStart = start + 2;
            el.selectionEnd = end + 2;
            el.focus();
            triggerAutoSave();
        }, 0);
    } else {
        const lineContent = value.substring(lineStart);
        let toRemove = 0;
        if (lineContent.startsWith("  ")) toRemove = 2;
        else if (lineContent.startsWith(" ")) toRemove = 1;
        if (toRemove > 0) {
            updateValue(
                value.substring(0, lineStart) +
                    value.substring(lineStart + toRemove),
            );
            setTimeout(() => {
                el.selectionStart = Math.max(lineStart, start - toRemove);
                el.selectionEnd = Math.max(lineStart, end - toRemove);
                el.focus();
                triggerAutoSave();
            }, 0);
        }
    }
};

// Xử lý phím tắt
const handleKeydown = (id, e, noteIndex = null) => {
    const refId = noteIndex !== null ? `notes_${noteIndex}` : id;
    const el = textareaRefs[refId];
    if (!el) return;
    const value =
        noteIndex !== null
            ? reportData.value.notes[noteIndex].content
            : reportData.value[id];

    const updateValue = (newValue) => {
        if (noteIndex !== null) {
            reportData.value.notes[noteIndex].content = newValue;
        } else {
            reportData.value[id] = newValue;
        }
    };

    const pairs = { "(": ")", "[": "]", "{": "}", '"': '"', "'": "'" };
    const closingChars = {
        ")": true,
        "]": true,
        "}": true,
        '"': true,
        "'": true,
    };

    if (closingChars[e.key]) {
        const start = el.selectionStart;
        if (value.substring(start, start + 1) === e.key) {
            e.preventDefault();
            el.selectionStart = el.selectionEnd = start + 1;
            return;
        }
    }

    if (pairs[e.key]) {
        e.preventDefault();
        const start = el.selectionStart;
        const end = el.selectionEnd;
        const closing = pairs[e.key];

        if (start !== end) {
            const selectedText = value.substring(start, end);
            updateValue(
                value.substring(0, start) +
                    e.key +
                    selectedText +
                    closing +
                    value.substring(end),
            );
            setTimeout(() => {
                el.selectionStart = start + 1;
                el.selectionEnd = end + 1;
                triggerAutoSave();
            }, 0);
        } else {
            updateValue(
                value.substring(0, start) +
                    e.key +
                    closing +
                    value.substring(start),
            );
            setTimeout(() => {
                el.selectionStart = el.selectionEnd = start + 1;
                triggerAutoSave();
            }, 0);
        }
        return;
    }

    if (e.key === "Enter") {
        const start = el.selectionStart;
        const beforeStart = value.substring(0, start);
        const lastNewLine = beforeStart.lastIndexOf("\n");
        const currentLine = beforeStart.substring(lastNewLine + 1);
        const match = currentLine.match(/^(\s*(?:[-+*]+|\d+\.)\s+)/);
        if (match) {
            e.preventDefault();
            const prefix = match[1];
            const afterStart = value.substring(start);
            updateValue(beforeStart + "\n" + prefix + afterStart);
            setTimeout(() => {
                el.selectionStart = el.selectionEnd = start + 1 + prefix.length;
                triggerAutoSave();
            }, 0);
        }
    } else if (e.key === "Tab") {
        e.preventDefault();
        adjustIndent(id, e.shiftKey ? -1 : 1, noteIndex);
    }
};

watch(
    reportData,
    () => {
        if (!isDeleting && saveStatus.value !== "saving") triggerAutoSave();
    },
    { deep: true },
);
watch(
    () => props.selectedDate,
    (newDate) => {
        loadReport(newDate);
    },
    { immediate: true },
);

const handleDataImported = () => {
    loadReport(props.selectedDate);
};

onMounted(() => {
    loadReport(props.selectedDate);
    window.addEventListener("data-imported", handleDataImported);
});

onUnmounted(() => {
    window.removeEventListener("data-imported", handleDataImported);
});
</script>

<template>
    <div class="daily-form">
        <header class="form-header">
            <div class="header-top">
                <div class="header-main">
                    <h1 class="header-date">{{ displayDate }}</h1>
                    <span v-if="isToday" class="today-label">Hôm nay</span>
                </div>
                <div class="save-indicator" :class="saveStatus">
                    <span v-if="saveStatus === 'saved'">
                        <svg
                            xmlns="http://www.w3.org/2000/svg"
                            width="14"
                            height="14"
                            viewBox="0 0 24 24"
                            fill="none"
                            stroke="currentColor"
                            stroke-width="3"
                            stroke-linecap="round"
                            stroke-linejoin="round"
                        >
                            <path d="M20 6 9 17l-5-5" />
                        </svg>
                        Đã lưu
                    </span>
                    <span v-if="saveStatus === 'saving'">Đang lưu...</span>
                    <span v-if="saveStatus === 'unsaved'">Chưa lưu</span>
                </div>
            </div>
            <p class="header-subtitle">
                {{
                    isToday
                        ? "Bản báo cáo công việc hàng ngày"
                        : "Xem lại báo cáo cũ"
                }}
            </p>
        </header>

        <div class="form-sections">
            <div
                v-for="section in sections"
                :key="section.id"
                class="section"
                :class="{
                    'is-focused': zoomedSection === section.id,
                    'is-collapsed': collapsedSections.includes(section.id),
                    'can-expand':
                        !collapsedSections.includes(section.id) &&
                        collapsedSections.length > 0,
                }"
            >
                <div class="section-meta">
                    <div
                        class="label-group"
                        @click="toggleCollapse(section.id)"
                    >
                        <span class="collapse-icon">
                            <svg
                                v-if="collapsedSections.includes(section.id)"
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="3"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <path d="m9 18 6-6-6-6" />
                            </svg>
                            <svg
                                v-else
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="3"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <path d="m6 9 6 6 6-6" />
                            </svg>
                        </span>
                        <label class="section-label">{{ section.label }}</label>
                    </div>
                    <div
                        class="section-actions"
                        v-if="!collapsedSections.includes(section.id)"
                    >
                        <button
                            class="icon-action-btn"
                            @click.stop="copySection(section.id, section.label)"
                            title="Sao chép nội dung phần này"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <rect
                                    width="14"
                                    height="14"
                                    x="8"
                                    y="8"
                                    rx="2"
                                    ry="2"
                                />
                                <path
                                    d="M4 16c-1.1 0-2-.9-2-2V4c0-1.1.9-2 2-2h10c1.1 0 2 .9 2 2"
                                />
                            </svg>
                        </button>
                        <button
                            class="icon-action-btn"
                            @click.stop="toggleZoom(section.id)"
                            :title="
                                zoomedSection === section.id
                                    ? 'Thu nhỏ'
                                    : 'Phóng to'
                            "
                        >
                            <svg
                                v-if="zoomedSection !== section.id"
                                xmlns="http://www.w3.org/2000/svg"
                                width="18"
                                height="18"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <path d="m15 3 6 6-6 6M9 21l-6-6 6-6" />
                            </svg>
                            <svg
                                v-else
                                xmlns="http://www.w3.org/2000/svg"
                                width="18"
                                height="18"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <path
                                    d="m21 21-6-6m6 0v6m0-6h-6M3 3l6 6M3 9V3m0 0h6"
                                />
                            </svg>
                        </button>
                    </div>
                </div>

                <div
                    v-show="!collapsedSections.includes(section.id)"
                    class="input-container"
                >
                    <template v-if="section.id !== 'notes'">
                        <textarea
                            :ref="(el) => setRef(section.id, el)"
                            v-model="reportData[section.id]"
                            :placeholder="section.placeholder"
                            @keydown="handleKeydown(section.id, $event)"
                        ></textarea>
                    </template>
                    <template v-else>
                        <div class="notes-container">
                            <div
                                v-for="(note, index) in reportData.notes"
                                :key="note.id"
                                class="sub-note-block"
                                :class="{
                                    'is-focused':
                                        zoomedSection === `notes_${index}`,
                                }"
                            >
                                <div class="sub-note-header">
                                    <input
                                        type="text"
                                        v-model="note.title"
                                        class="sub-note-title"
                                        placeholder="Tiêu đề ghi chú..."
                                    />
                                    <div class="sub-note-actions">
                                        <button
                                            class="icon-action-btn"
                                            @click.stop="copySubNote(index)"
                                            title="Sao chép ghi chú này"
                                        >
                                            <svg
                                                xmlns="http://www.w3.org/2000/svg"
                                                width="16"
                                                height="16"
                                                viewBox="0 0 24 24"
                                                fill="none"
                                                stroke="currentColor"
                                                stroke-width="2"
                                                stroke-linecap="round"
                                                stroke-linejoin="round"
                                            >
                                                <rect
                                                    width="14"
                                                    height="14"
                                                    x="8"
                                                    y="8"
                                                    rx="2"
                                                    ry="2"
                                                />
                                                <path
                                                    d="M4 16c-1.1 0-2-.9-2-2V4c0-1.1.9-2 2-2h10c1.1 0 2 .9 2 2"
                                                />
                                            </svg>
                                        </button>
                                        <button
                                            class="icon-action-btn"
                                            @click.stop="
                                                toggleZoom(`notes_${index}`)
                                            "
                                            :title="
                                                zoomedSection ===
                                                `notes_${index}`
                                                    ? 'Thu nhỏ'
                                                    : 'Phóng to'
                                            "
                                        >
                                            <svg
                                                v-if="
                                                    zoomedSection !==
                                                    `notes_${index}`
                                                "
                                                xmlns="http://www.w3.org/2000/svg"
                                                width="18"
                                                height="18"
                                                viewBox="0 0 24 24"
                                                fill="none"
                                                stroke="currentColor"
                                                stroke-width="2"
                                                stroke-linecap="round"
                                                stroke-linejoin="round"
                                            >
                                                <path
                                                    d="m15 3 6 6-6 6M9 21l-6-6 6-6"
                                                />
                                            </svg>
                                            <svg
                                                v-else
                                                xmlns="http://www.w3.org/2000/svg"
                                                width="18"
                                                height="18"
                                                viewBox="0 0 24 24"
                                                fill="none"
                                                stroke="currentColor"
                                                stroke-width="2"
                                                stroke-linecap="round"
                                                stroke-linejoin="round"
                                            >
                                                <path
                                                    d="m21 21-6-6m6 0v6m0-6h-6M3 3l6 6M3 9V3m0 0h6"
                                                />
                                            </svg>
                                        </button>
                                        <button
                                            class="icon-action-btn delete-note-btn"
                                            @click="removeNote(index)"
                                            title="Xóa ghi chú này"
                                        >
                                            <svg
                                                xmlns="http://www.w3.org/2000/svg"
                                                width="16"
                                                height="16"
                                                viewBox="0 0 24 24"
                                                fill="none"
                                                stroke="currentColor"
                                                stroke-width="2"
                                                stroke-linecap="round"
                                                stroke-linejoin="round"
                                            >
                                                <path d="M3 6h18" />
                                                <path
                                                    d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"
                                                />
                                                <path
                                                    d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"
                                                />
                                                <line
                                                    x1="10"
                                                    x2="10"
                                                    y1="11"
                                                    y2="17"
                                                />
                                                <line
                                                    x1="14"
                                                    x2="14"
                                                    y1="11"
                                                    y2="17"
                                                />
                                            </svg>
                                        </button>
                                    </div>
                                </div>
                                <textarea
                                    :ref="(el) => setRef(`notes_${index}`, el)"
                                    v-model="note.content"
                                    placeholder="Nội dung ghi chú..."
                                    @keydown="
                                        handleKeydown('notes', $event, index)
                                    "
                                ></textarea>
                                <button
                                    v-if="zoomedSection === `notes_${index}`"
                                    class="close-focus-btn"
                                    @click="toggleZoom(`notes_${index}`)"
                                >
                                    Hoàn tất
                                </button>
                            </div>
                            <button
                                class="btn-outline add-note-btn"
                                @click="addNote"
                            >
                                + Thêm Ghi Chú
                            </button>
                        </div>
                    </template>

                    <button
                        v-if="zoomedSection === section.id"
                        class="close-focus-btn"
                        @click="toggleZoom(section.id)"
                    >
                        Hoàn tất
                    </button>
                </div>
            </div>
        </div>

        <div
            v-if="zoomedSection"
            class="focus-overlay"
            @click="toggleZoom(zoomedSection)"
        ></div>

        <footer class="form-footer">
            <button
                class="btn-danger-outline"
                @click="confirmDelete"
                title="Xóa báo cáo"
            >
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="18"
                    height="18"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                >
                    <path d="M3 6h18" />
                    <path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6" />
                    <path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2" />
                    <line x1="10" x2="10" y1="11" y2="17" />
                    <line x1="14" x2="14" y1="11" y2="17" />
                </svg>
            </button>
            <button class="btn-primary" @click="saveReport">
                Cập nhật báo cáo
            </button>
        </footer>

        <!-- Custom Confirm Modal -->
        <Teleport to="body">
            <div v-if="showConfirmModal" class="modal-overlay">
                <div class="modal-content">
                    <div class="modal-icon">
                        <svg
                            xmlns="http://www.w3.org/2000/svg"
                            width="32"
                            height="32"
                            viewBox="0 0 24 24"
                            fill="none"
                            stroke="currentColor"
                            stroke-width="2"
                            stroke-linecap="round"
                            stroke-linejoin="round"
                        >
                            <path
                                d="m21.73 18-8-14a2 2 0 0 0-3.48 0l-8 14A2 2 0 0 0 4 21h16a2 2 0 0 0 1.73-3Z"
                            />
                            <line x1="12" x2="12" y1="9" y2="13" />
                            <line x1="12" x2="12.01" y1="17" y2="17" />
                        </svg>
                    </div>
                    <h3>Xác nhận xóa</h3>
                    <p>
                        Bạn có chắc chắn muốn xóa báo cáo của ngày
                        <strong>{{ displayDate }}</strong> không? Thao tác này
                        không thể hoàn tác.
                    </p>
                    <div class="modal-actions">
                        <button
                            class="btn-outline"
                            @click="showConfirmModal = false"
                        >
                            Hủy
                        </button>
                        <button class="btn-danger" @click="executeDelete">
                            Xóa báo cáo
                        </button>
                    </div>
                </div>
            </div>
        </Teleport>

        <!-- Custom Confirm Modal cho Ghi chú con -->
        <Teleport to="body">
            <div v-if="showDeleteNoteModal" class="modal-overlay">
                <div class="modal-content">
                    <div class="modal-icon">
                        <svg
                            xmlns="http://www.w3.org/2000/svg"
                            width="32"
                            height="32"
                            viewBox="0 0 24 24"
                            fill="none"
                            stroke="currentColor"
                            stroke-width="2"
                            stroke-linecap="round"
                            stroke-linejoin="round"
                        >
                            <path
                                d="m21.73 18-8-14a2 2 0 0 0-3.48 0l-8 14A2 2 0 0 0 4 21h16a2 2 0 0 0 1.73-3Z"
                            />
                            <line x1="12" x2="12" y1="9" y2="13" />
                            <line x1="12" x2="12.01" y1="17" y2="17" />
                        </svg>
                    </div>
                    <h3>Xác nhận xóa ghi chú</h3>
                    <p>
                        Bạn có chắc chắn muốn xóa ghi chú này không? Thao tác
                        này không thể hoàn tác.
                    </p>
                    <div class="modal-actions">
                        <button
                            class="btn-outline"
                            @click="showDeleteNoteModal = false"
                        >
                            Hủy
                        </button>
                        <button class="btn-danger" @click="executeRemoveNote">
                            Xóa ghi chú
                        </button>
                    </div>
                </div>
            </div>
        </Teleport>
    </div>
</template>

<style scoped>
.daily-form {
    display: flex;
    flex-direction: column;
    gap: 40px;
    position: relative;
}

.form-header {
    border-bottom: 1px solid var(--border-color);
    padding-bottom: 24px;
}
.header-top {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 4px;
}
.header-main {
    display: flex;
    align-items: center;
    gap: 12px;
}
.header-date {
    font-size: 2.5rem;
    letter-spacing: -0.03em;
}
.today-label {
    background-color: var(--bg-secondary);
    color: var(--color-primary);
    font-size: 0.8rem;
    font-weight: 800;
    padding: 4px 10px;
    border-radius: 20px;
    text-transform: uppercase;
}
.header-subtitle {
    font-size: 1.1rem;
    color: var(--text-light);
}

.save-indicator {
    font-size: 0.75rem;
    font-weight: 700;
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 4px 10px;
    border-radius: 6px;
    transition: var(--transition-fast);
}
.save-indicator.saved {
    color: #4caf50;
    background-color: rgba(76, 175, 80, 0.1);
}
.save-indicator.saving {
    color: var(--color-primary);
    background-color: var(--bg-primary);
}
.save-indicator.unsaved {
    color: #ffa000;
    background-color: rgba(255, 160, 0, 0.1);
}

.form-sections {
    display: flex;
    flex-direction: column;
    gap: 24px;
}
.section {
    transition: var(--transition-fast);
}

.section-meta {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 8px;
}

.label-group {
    display: flex;
    align-items: center;
    gap: 8px;
    cursor: pointer;
    padding: 4px;
    border-radius: 6px;
    transition: var(--transition-fast);
}
.label-group:hover {
    background-color: var(--bg-secondary);
}

.collapse-icon {
    display: flex;
    align-items: center;
    color: var(--text-light);
}

.section-label {
    font-size: 0.85rem;
    font-weight: 800;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: var(--text-medium);
    pointer-events: none;
}

.section-actions {
    display: flex;
    gap: 4px;
}

.icon-action-btn {
    background: transparent;
    color: var(--text-light);
    opacity: 0.5;
    padding: 4px;
    border-radius: 4px;
    display: flex;
    align-items: center;
    justify-content: center;
}
.icon-action-btn:hover {
    opacity: 1;
    color: var(--color-primary);
    background: var(--bg-secondary);
}

.input-container {
    background-color: var(--bg-secondary);
    border: 1.5px solid transparent;
    border-radius: 12px;
    padding: 16px;
    transition: var(--transition-fast);
    position: relative;
}
.input-container:focus-within {
    background-color: #ffffff;
    border-color: var(--color-primary);
}

.notes-container {
    display: flex;
    flex-direction: column;
    gap: 16px;
}

.sub-note-block {
    background-color: #fff;
    border: 1px solid var(--border-color);
    border-radius: 8px;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sub-note-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid var(--bg-secondary);
    padding-bottom: 8px;
}

.sub-note-actions {
    display: flex;
    gap: 4px;
}

.sub-note-block.is-focused {
    position: fixed;
    inset: 40px;
    z-index: 2200;
    display: flex;
    flex-direction: column;
    gap: 20px;
    padding: 30px;
    box-shadow: 0 20px 50px rgba(0, 0, 0, 0.1);
    background-color: #ffffff;
    animation: zoomIn 0.3s ease-out;
}
.sub-note-block.is-focused textarea {
    flex: 1;
    font-size: 1.2rem;
    min-height: unset !important;
}
.sub-note-block.is-focused .sub-note-title {
    font-size: 1.5rem;
    color: var(--color-primary);
}

.sub-note-title {
    font-size: 1rem;
    font-weight: 700;
    color: var(--text-dark);
    background: transparent;
    border: none;
    outline: none;
    width: 100%;
}

.delete-note-btn {
    color: var(--text-light);
}

.delete-note-btn:hover {
    color: #f44336;
    background: rgba(244, 67, 54, 0.1);
}

.add-note-btn {
    border: 1px dashed var(--color-primary);
    color: var(--color-primary);
    padding: 10px;
    border-radius: 8px;
    background: transparent;
    font-weight: 600;
    cursor: pointer;
    transition: var(--transition-fast);
}

.add-note-btn:hover {
    background: var(--bg-secondary);
}

textarea {
    width: 100%;
    min-height: 120px;
    font-size: 1.05rem;
    line-height: 1.6;
    transition: min-height 0.3s ease;
    padding-right: 8px;
}

/* Tự động dài ra khi có phần khác bị thu gọn */
.section.can-expand textarea {
    min-height: 200px;
}

/* Focus Mode Styles */
.section.is-focused {
    position: fixed;
    inset: 40px;
    z-index: 2100;
    display: flex;
    flex-direction: column;
    gap: 20px;
    animation: zoomIn 0.3s ease-out;
}
.section.is-focused .input-container {
    flex: 1;
    display: flex;
    flex-direction: column;
    padding: 30px;
    box-shadow: 0 20px 50px rgba(0, 0, 0, 0.1);
    background-color: #ffffff;
}
.section.is-focused .input-container > textarea {
    flex: 1;
    font-size: 1.2rem;
    min-height: unset !important;
}

.section.is-focused .notes-container {
    flex: 1;
    overflow-y: auto;
    padding-right: 8px;
}

.section.is-focused .sub-note-block textarea {
    min-height: 250px;
    font-size: 1.15rem;
}

.section.is-focused .section-label {
    font-size: 1.5rem;
    color: var(--color-primary);
}

.close-focus-btn {
    position: absolute;
    bottom: 20px;
    right: 20px;
    background-color: var(--color-primary);
    color: white;
    padding: 10px 20px;
    border-radius: 8px;
    font-weight: 700;
}

.focus-overlay {
    position: fixed;
    inset: 0;
    background: rgba(255, 251, 241, 0.9);
    backdrop-filter: blur(5px);
    z-index: 2000;
    animation: fadeIn 0.3s ease-out;
}

@keyframes zoomIn {
    from {
        opacity: 0;
        transform: scale(0.95);
    }
    to {
        opacity: 1;
        transform: scale(1);
    }
}
@keyframes fadeIn {
    from {
        opacity: 0;
    }
    to {
        opacity: 1;
    }
}

.form-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 16px;
    padding-top: 16px;
}
.form-footer button {
    height: 46px;
    display: flex;
    align-items: center;
    justify-content: center;
}

@media (max-width: 900px) {
    .form-footer {
        flex-direction: column;
        align-items: flex-end;
    }
}
@media (max-width: 640px) {
    .header-date {
        font-size: 2rem;
    }
    .header-top {
        flex-direction: column;
        align-items: flex-start;
        gap: 8px;
    }
    .form-footer {
        flex-direction: column;
        width: 100%;
        align-items: stretch;
    }
    .form-footer button {
        height: 50px;
    }
    .section.is-focused {
        inset: 10px;
        padding: 10px;
    }
}

/* Modal Styles */
.modal-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.4);
    backdrop-filter: blur(2px);
    z-index: 3000;
    display: flex;
    align-items: center;
    justify-content: center;
}

.modal-content {
    background-color: #ffffff;
    border-radius: 12px;
    padding: 30px;
    max-width: 400px;
    width: 90%;
    text-align: center;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
}

.modal-icon {
    color: #f44336;
    margin-bottom: 16px;
    display: flex;
    justify-content: center;
}

.modal-content h3 {
    margin-bottom: 12px;
    font-size: 1.2rem;
}

.modal-content p {
    color: var(--text-medium);
    font-size: 0.95rem;
    margin-bottom: 24px;
}

.modal-actions {
    display: flex;
    justify-content: center;
    gap: 12px;
}

.btn-danger-outline {
    background-color: transparent;
    color: #f44336;
    border: 1.5px solid #f44336;
    padding: 10px 14px;
    border-radius: 8px;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: var(--transition-fast);
}

.btn-danger-outline:hover {
    background-color: rgba(244, 67, 54, 0.1);
}

.btn-danger {
    background-color: #f44336;
    color: white;
    padding: 10px 22px;
    font-weight: 600;
    border-radius: 8px;
    transition: var(--transition-fast);
}

.btn-danger:hover {
    filter: brightness(1.1);
}
</style>
