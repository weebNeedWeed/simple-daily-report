<script setup>
import {
    ref,
    onMounted,
    onUnmounted,
    defineProps,
    defineEmits,
    watch,
    nextTick,
} from "vue";

const props = defineProps(["selectedDate"]);
const emit = defineEmits(["select-date"]);

const historyDates = ref([]);
const fileInput = ref(null);
const dateInputRef = ref(null);

const showCreateModal = ref(false);
const pendingDate = ref(null);

const triggerDatePicker = () => {
    if (dateInputRef.value) {
        if (typeof dateInputRef.value.showPicker === "function") {
            dateInputRef.value.showPicker();
        } else {
            dateInputRef.value.click();
        }
    }
};

const handleCustomDateSelect = (event) => {
    const dateStr = event.target.value;
    if (!dateStr) return;

    if (historyDates.value.includes(dateStr)) {
        emit("select-date", dateStr);
    } else {
        pendingDate.value = dateStr;
        showCreateModal.value = true;
    }

    if (dateInputRef.value) dateInputRef.value.value = "";
};

const confirmCreateNew = () => {
    if (pendingDate.value) {
        emit("select-date", pendingDate.value);
    }
    showCreateModal.value = false;
    pendingDate.value = null;
};

// Thông báo (Dispatch global toast)
const showToast = (message, type = "success") => {
    window.dispatchEvent(
        new CustomEvent("show-toast", { detail: { message, type } }),
    );
};

// Tải danh sách lịch sử từ LocalStorage
const loadHistory = () => {
    const saved = localStorage.getItem("dr_history");
    if (saved) {
        try {
            historyDates.value = JSON.parse(saved);
        } catch (e) {
            console.error("Lỗi khi tải lịch sử:", e);
        }
    }
};

// Xuất dữ liệu ra file JSON
const exportData = () => {
    const data = {};
    // Lấy danh sách lịch sử
    const history = localStorage.getItem("dr_history");
    if (history) data["dr_history"] = JSON.parse(history);

    // Lấy dữ liệu của từng ngày
    Object.keys(localStorage).forEach((key) => {
        if (key.startsWith("dr_data_")) {
            data[key] = JSON.parse(localStorage.getItem(key));
        }
    });

    if (Object.keys(data).length === 0) {
        showToast("Không có dữ liệu để xuất!", "error");
        return;
    }

    const blob = new Blob([JSON.stringify(data, null, 2)], {
        type: "application/json",
    });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `daily-report-backup-${new Date().toISOString().split("T")[0]}.json`;
    a.click();
    URL.revokeObjectURL(url);
    showToast("Đã xuất dữ liệu thành công!");
};

// Nhập dữ liệu từ file JSON
const handleImport = (event) => {
    const file = event.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
        try {
            const data = JSON.parse(e.target.result);

            // Validate sơ bộ
            if (!data.dr_history) {
                throw new Error("File không đúng định dạng!");
            }

            // Ghi dữ liệu vào localStorage
            Object.keys(data).forEach((key) => {
                localStorage.setItem(key, JSON.stringify(data[key]));
            });

            loadHistory();
            showToast("Đã phục hồi dữ liệu thành công!");

            // Thông báo cho Form để tải lại dữ liệu ngày hiện tại
            window.dispatchEvent(new CustomEvent("data-imported"));
        } catch (err) {
            showToast("Lỗi: " + err.message, "error");
        }
    };
    reader.readAsText(file);
    event.target.value = ""; // Reset input
};

const triggerImport = () => {
    fileInput.value.click();
};

const handleReportSaved = () => {
    loadHistory();
};

const handleRequestNewDate = (e) => {
    const dateStr = e.detail;
    pendingDate.value = dateStr;
    showCreateModal.value = true;
};

// Tự động cuộn đến thẻ đang active
const scrollToActiveCard = () => {
    nextTick(() => {
        if (!props.selectedDate) return;
        const card = document.getElementById("card-" + props.selectedDate);
        if (card) {
            // Cuộn thẻ vào khu vực nhìn thấy
            card.scrollIntoView({ behavior: "smooth", block: "center" });
        }
    });
};

const handleGlobalChangeDate = (e) => {
    if (e.detail === props.selectedDate) {
        scrollToActiveCard();
    }
};

onMounted(() => {
    loadHistory();
    window.addEventListener("report-saved", handleReportSaved);
    window.addEventListener("trigger-export", exportData);
    window.addEventListener("request-new-date", handleRequestNewDate);
    window.addEventListener("change-date", handleGlobalChangeDate);
});

onUnmounted(() => {
    window.removeEventListener("report-saved", handleReportSaved);
    window.removeEventListener("trigger-export", exportData);
    window.removeEventListener("request-new-date", handleRequestNewDate);
    window.removeEventListener("change-date", handleGlobalChangeDate);
});

const formatDate = (dateStr) => {
    const date = new Date(dateStr);
    return date.toLocaleDateString("vi-VN", {
        day: "2-digit",
        month: "2-digit",
        year: "numeric",
    });
};

const selectDate = (date) => {
    emit("select-date", date);
    if (date === props.selectedDate) {
        scrollToActiveCard();
    }
};

const isToday = (dateStr) => {
    const today = new Date();
    const todayStr = `${today.getFullYear()}-${String(today.getMonth() + 1).padStart(2, "0")}-${String(today.getDate()).padStart(2, "0")}`;
    return dateStr === todayStr;
};

watch(
    () => props.selectedDate,
    () => {
        scrollToActiveCard();
    },
    { immediate: true },
);

watch(historyDates, () => {
    scrollToActiveCard();
});

const dispatchOpenLauncher = () => {
    window.dispatchEvent(new CustomEvent("open-launcher"));
};
</script>

<template>
    <div class="history-panel">
        <div class="history-header">
            <h2 class="history-title">Lịch sử</h2>
            <div class="header-actions">
                <button
                    class="icon-btn search-hint-btn"
                    title="Tìm kiếm (Ctrl + K)"
                    @click="dispatchOpenLauncher"
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
                        <circle cx="11" cy="11" r="8" />
                        <path d="m21 21-4.3-4.3" />
                    </svg>
                    <span class="hint-text">Ctrl K</span>
                </button>
                <input
                    type="date"
                    class="hidden-date-input"
                    ref="dateInputRef"
                    @change="handleCustomDateSelect"
                />
                <button
                    class="icon-btn"
                    title="Thêm ngày mới"
                    @click="triggerDatePicker"
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
                            width="18"
                            height="18"
                            x="3"
                            y="4"
                            rx="2"
                            ry="2"
                        />
                        <line x1="16" x2="16" y1="2" y2="6" />
                        <line x1="8" x2="8" y1="2" y2="6" />
                        <line x1="3" x2="21" y1="10" y2="10" />
                        <path d="M12 14v4" />
                        <path d="M10 16h4" />
                    </svg>
                </button>
                <button
                    class="new-btn"
                    @click="
                        selectDate(
                            `${new Date().getFullYear()}-${String(new Date().getMonth() + 1).padStart(2, '0')}-${String(new Date().getDate()).padStart(2, '0')}`,
                        )
                    "
                >
                    Hôm nay
                </button>
            </div>
        </div>

        <div class="history-items">
            <div v-if="historyDates.length === 0" class="empty-history">
                Chưa có báo cáo nào được lưu.
            </div>
            <div
                v-for="date in historyDates"
                :key="date"
                :id="'card-' + date"
                class="history-card"
                :class="{ active: props.selectedDate === date }"
                @click="selectDate(date)"
            >
                <div class="history-info">
                    <div class="date-group">
                        <span class="history-date">{{ formatDate(date) }}</span>
                        <span v-if="isToday(date)" class="today-tag"
                            >Hiện tại</span
                        >
                    </div>
                    <span class="history-badge">Đã lưu</span>
                </div>
            </div>
        </div>

        <!-- Khu vực Sao lưu & Phục hồi -->
        <div class="backup-section">
            <h3 class="backup-title">Sao lưu & Phục hồi</h3>
            <div class="backup-actions">
                <button class="backup-btn" @click="exportData">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="14"
                        height="14"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
                        <polyline points="7 10 12 15 17 10" />
                        <line x1="12" x2="12" y1="15" y2="3" />
                    </svg>
                    Xuất JSON
                </button>
                <button class="backup-btn" @click="triggerImport">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="14"
                        height="14"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
                        <polyline points="17 8 12 3 7 8" />
                        <line x1="12" x2="12" y1="3" y2="15" />
                    </svg>
                    Nhập JSON
                </button>
                <input
                    type="file"
                    ref="fileInput"
                    accept=".json"
                    hidden
                    @change="handleImport"
                />
            </div>
        </div>
        <!-- Custom Confirm Modal -->
        <Teleport to="body">
            <div v-if="showCreateModal" class="modal-overlay">
                <div class="modal-content">
                    <div class="modal-icon-primary">
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
                                d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"
                            />
                            <polyline points="14 2 14 8 20 8" />
                            <line x1="12" y1="18" x2="12" y2="12" />
                            <line x1="9" y1="15" x2="15" y2="15" />
                        </svg>
                    </div>
                    <h3>Tạo báo cáo mới?</h3>
                    <p>
                        Ngày <strong>{{ formatDate(pendingDate) }}</strong> chưa
                        có báo cáo. Bạn có muốn tạo mới không?
                    </p>
                    <div class="modal-actions">
                        <button
                            class="btn-outline"
                            @click="showCreateModal = false"
                        >
                            Hủy
                        </button>
                        <button class="btn-primary" @click="confirmCreateNew">
                            Tạo mới
                        </button>
                    </div>
                </div>
            </div>
        </Teleport>
    </div>
</template>

<style scoped>
.history-panel {
    display: flex;
    flex-direction: column;
    gap: 20px;
    height: 100%;
}

.history-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.header-actions {
    display: flex;
    gap: 8px;
    align-items: center;
    position: relative;
}

.hidden-date-input {
    width: 1px;
    height: 1px;
    opacity: 0;
    position: absolute;
    left: -100px;
    right: auto;
    bottom: -5px;
    border: none;
    padding: 0;
    margin: 0;
    pointer-events: none;
}

.icon-btn {
    background-color: transparent;
    border: 1px solid var(--border-color);
    color: var(--text-medium);
    padding: 6px;
    border-radius: 8px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    transition: var(--transition-fast);
}

.icon-btn:hover {
    border-color: var(--color-primary);
    color: var(--color-primary);
    background-color: var(--bg-primary);
}

.search-hint-btn {
    padding: 6px 8px;
    gap: 6px;
    font-size: 0.8rem;
    font-weight: 600;
}

.hint-text {
    opacity: 0.7;
    letter-spacing: 0.02em;
}

.history-title {
    font-size: 1.5rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: var(--text-dark);
}

.new-btn {
    background-color: var(--color-primary);
    color: white;
    padding: 6px 12px;
    border-radius: 8px;
    font-size: 0.8rem;
    font-weight: 700;
}

.history-items {
    display: flex;
    flex-direction: column;
    gap: 12px;
    max-height: calc(100vh - 350px);
    overflow-y: auto;
    padding-right: 4px;
    flex: 1;
}

.empty-history {
    color: var(--text-light);
    font-size: 0.9rem;
    padding: 20px;
    text-align: center;
    border: 1px dashed var(--border-color);
    border-radius: 12px;
    background-color: white;
}

.history-card {
    background-color: #ffffff;
    border: 1px solid var(--border-color);
    border-radius: 12px;
    padding: 16px;
    cursor: pointer;
    transition: var(--transition-fast);
}

.history-card:hover {
    border-color: var(--color-primary);
    background-color: var(--bg-primary);
}

.history-card.active {
    border-color: var(--color-primary);
    border-width: 2px;
    background-color: var(--bg-secondary);
}

.history-info {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.date-group {
    display: flex;
    align-items: center;
    gap: 8px;
}

.history-date {
    font-weight: 600;
    font-size: 0.95rem;
    color: var(--text-dark);
}

.today-tag {
    font-size: 0.65rem;
    background-color: var(--color-accent);
    color: white;
    padding: 2px 6px;
    border-radius: 4px;
    font-weight: 800;
}

.history-badge {
    font-size: 0.75rem;
    font-weight: 700;
    text-transform: uppercase;
    color: var(--text-light);
    letter-spacing: 0.05em;
}

/* Backup Section Styles */
.backup-section {
    margin-top: auto;
    padding-top: 20px;
    border-top: 1px solid var(--border-color);
}

.backup-title {
    font-size: 0.8rem;
    font-weight: 800;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--text-light);
    margin-bottom: 12px;
}

.backup-actions {
    display: flex;
    gap: 10px;
}

.backup-btn {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    background-color: #ffffff;
    border: 1px solid var(--border-color);
    color: var(--text-medium);
    padding: 10px;
    border-radius: 8px;
    font-size: 0.85rem;
    font-weight: 600;
    transition: var(--transition-fast);
}

.backup-btn:hover {
    border-color: var(--color-primary);
    color: var(--color-primary);
    background-color: var(--bg-primary);
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

.modal-icon-primary {
    color: var(--color-primary);
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

/* Animations */
.fadeIn-enter-active,
.fadeIn-leave-active {
    transition:
        opacity 0.2s ease,
        transform 0.2s ease;
}
.fadeIn-enter-from,
.fadeIn-leave-to {
    opacity: 0;
    transform: scale(0.95);
}
</style>
