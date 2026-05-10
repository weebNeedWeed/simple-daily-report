<script setup>
import { ref, onMounted, defineProps, watch, computed } from "vue";

const props = defineProps(["selectedDate"]);

const sections = [
    {
        id: "completed",
        label: "Việc đã hoàn thành",
        placeholder: "Những công việc bạn đã hoàn thành trong hôm nay...",
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
    next: "",
    blockers: "",
});

// Trạng thái lưu
const saveStatus = ref("saved"); // 'saved', 'saving', 'unsaved'

// Trạng thái phóng to (Focus Mode)
const zoomedSection = ref(null); // ID của section đang được phóng to

// Trạng thái thu gọn (Collapse) - Mặc định thu gọn phần blockers
const collapsedSections = ref(["blockers"]);

// Thông báo (Toast)
const toast = ref({ show: false, message: "", type: "success" });
const showToast = (msg, type = "success") => {
    toast.value = { show: true, message: msg, type };
    setTimeout(() => {
        toast.value.show = false;
    }, 3000);
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
    reportData.value = { completed: "", next: "", blockers: "" };
    const saved = localStorage.getItem(`dr_data_${dateKey}`);
    if (saved) {
        try {
            reportData.value = JSON.parse(saved);
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
const toggleZoom = (id) => {
    if (zoomedSection.value === id) {
        zoomedSection.value = null;
        document.body.style.overflow = "";
    } else {
        zoomedSection.value = id;
        document.body.style.overflow = "hidden";
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
const copyFullReport = () => {
    const content = `[BÁO CÁO NGÀY ${displayDate.value}]\n\n* VIỆC ĐÃ HOÀN THÀNH:\n${reportData.value.completed || "(Trống)"}\n\n* KẾ HOẠCH TIẾP THEO:\n${reportData.value.next || "(Trống)"}\n\n* KHÓ KHĂN (BLOCKERS):\n${reportData.value.blockers || "(Trống)"}`;
    navigator.clipboard
        .writeText(content)
        .then(() => {
            showToast("Đã sao chép toàn bộ báo cáo!");
        })
        .catch(() => {
            showToast("Không thể sao chép!", "error");
        });
};

const copyCompletedOnly = () => {
    const content = `* VIỆC ĐÃ HOÀN THÀNH (${displayDate.value}):\n${reportData.value.completed || "(Trống)"}`;
    navigator.clipboard
        .writeText(content)
        .then(() => {
            showToast("Đã sao chép việc đã hoàn thành!");
        })
        .catch(() => {
            showToast("Không thể sao chép!", "error");
        });
};

// Xử lý thụt lề
const adjustIndent = (id, amount) => {
    const el = textareaRefs[id];
    if (!el) return;
    const start = el.selectionStart;
    const end = el.selectionEnd;
    const value = reportData.value[id];
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
            reportData.value[id] =
                value.substring(0, lineStart) +
                newPrefix +
                afterLineStart.substring(currentMatch.length);
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
        reportData.value[id] =
            value.substring(0, lineStart) + "  " + value.substring(lineStart);
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
            reportData.value[id] =
                value.substring(0, lineStart) +
                value.substring(lineStart + toRemove);
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
const handleKeydown = (id, e) => {
    if (e.key === "Enter") {
        const el = textareaRefs[id];
        if (!el) return;
        const start = el.selectionStart;
        const value = reportData.value[id];
        const beforeStart = value.substring(0, start);
        const lastNewLine = beforeStart.lastIndexOf("\n");
        const currentLine = beforeStart.substring(lastNewLine + 1);
        const match = currentLine.match(/^(\s*(?:[-+*]+|\d+\.)\s+)/);
        if (match) {
            e.preventDefault();
            const prefix = match[1];
            if (currentLine === prefix) {
                reportData.value[id] =
                    value.substring(
                        0,
                        lastNewLine + (lastNewLine === -1 ? 0 : 1),
                    ) + value.substring(start);
                setTimeout(() => {
                    el.selectionStart = el.selectionEnd =
                        lastNewLine + (lastNewLine === -1 ? 0 : 1);
                    triggerAutoSave();
                }, 0);
                return;
            }
            const afterStart = value.substring(start);
            reportData.value[id] = beforeStart + "\n" + prefix + afterStart;
            setTimeout(() => {
                el.selectionStart = el.selectionEnd = start + 1 + prefix.length;
                triggerAutoSave();
            }, 0);
        }
    } else if (e.key === "Tab") {
        e.preventDefault();
        adjustIndent(id, e.shiftKey ? -1 : 1);
    }
};

watch(
    reportData,
    () => {
        if (saveStatus.value !== "saving") triggerAutoSave();
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

onMounted(() => {
    loadReport(props.selectedDate);
});
</script>

<template>
    <div class="daily-form">
        <Transition name="fade">
            <div v-if="toast.show" class="toast" :class="toast.type">
                {{ toast.message }}
            </div>
        </Transition>

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
                    <button
                        class="zoom-btn"
                        v-if="!collapsedSections.includes(section.id)"
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

                <div
                    v-show="!collapsedSections.includes(section.id)"
                    class="input-container"
                >
                    <textarea
                        :ref="(el) => setRef(section.id, el)"
                        v-model="reportData[section.id]"
                        :placeholder="section.placeholder"
                        @keydown="handleKeydown(section.id, $event)"
                    ></textarea>
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
            <div class="copy-options">
                <button class="btn-outline" @click="copyCompletedOnly">
                    Sao chép việc đã xong
                </button>
                <button class="btn-outline" @click="copyFullReport">
                    Sao chép toàn bộ
                </button>
            </div>
            <button class="btn-primary" @click="saveReport">
                Cập nhật báo cáo
            </button>
        </footer>
    </div>
</template>

<style scoped>
.daily-form {
    display: flex;
    flex-direction: column;
    gap: 40px;
    position: relative;
}

.toast {
    position: fixed;
    top: 20px;
    right: 20px;
    padding: 12px 24px;
    border-radius: 8px;
    color: white;
    font-weight: 600;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    z-index: 3000;
}
.toast.success {
    background-color: #4caf50;
}
.toast.error {
    background-color: #f44336;
}

.fade-enter-active,
.fade-leave-active {
    transition:
        opacity 0.3s,
        transform 0.3s;
}
.fade-enter-from,
.fade-leave-to {
    opacity: 0;
    transform: translateY(-20px);
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

.zoom-btn {
    background: transparent;
    color: var(--text-light);
    opacity: 0.5;
    padding: 4px;
    border-radius: 4px;
    display: flex;
    align-items: center;
    justify-content: center;
}
.zoom-btn:hover {
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
.section.is-focused textarea {
    flex: 1;
    font-size: 1.2rem;
    min-height: unset !important;
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
.copy-options {
    display: flex;
    gap: 12px;
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
    .form-footer,
    .copy-options {
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
</style>
