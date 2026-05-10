<script setup>
import { ref, onMounted, onUnmounted, defineProps, defineEmits } from "vue";

const props = defineProps(["selectedDate"]);
const emit = defineEmits(["select-date"]);

const historyDates = ref([]);
const fileInput = ref(null);

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
            window.dispatchEvent(new CustomEvent("report-saved"));
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

onMounted(() => {
    loadHistory();
    window.addEventListener("report-saved", handleReportSaved);
});

onUnmounted(() => {
    window.removeEventListener("report-saved", handleReportSaved);
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
};

const isToday = (dateStr) => {
    const today = new Date();
    const todayStr = `${today.getFullYear()}-${String(today.getMonth() + 1).padStart(2, "0")}-${String(today.getDate()).padStart(2, "0")}`;
    return dateStr === todayStr;
};
</script>

<template>
    <div class="history-panel">
        <div class="history-header">
            <h2 class="history-title">Lịch sử</h2>
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

        <div class="history-items">
            <div v-if="historyDates.length === 0" class="empty-history">
                Chưa có báo cáo nào được lưu.
            </div>
            <div
                v-for="date in historyDates"
                :key="date"
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
</style>
