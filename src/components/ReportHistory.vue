<script setup>
import { ref, onMounted, onUnmounted, defineProps, defineEmits } from "vue";

const props = defineProps(["selectedDate"]);
const emit = defineEmits(["select-date"]);

const historyDates = ref([]);

// Tải danh sách lịch sử từ LocalStorage
const loadHistory = () => {
    const saved = localStorage.getItem("dr_history");
    if (saved) {
        try {
            historyDates.value = JSON.parse(saved);

            // Đảm bảo ngày hiện tại luôn có trong danh sách hiển thị để chọn
            const todayKey = new Date().toISOString().split("T")[0];
            const todayKeyLocal = `${new Date().getFullYear()}-${String(new Date().getMonth() + 1).padStart(2, "0")}-${String(new Date().getDate()).padStart(2, "0")}`;

            if (!historyDates.value.includes(todayKeyLocal)) {
                // Chúng ta không lưu vào LocalStorage ở đây, chỉ hiển thị ở UI
                // Nhưng thường người dùng muốn quay lại ngày hôm nay
            }
        } catch (e) {
            console.error("Lỗi khi tải lịch sử:", e);
        }
    }
};

// Lắng nghe sự kiện lưu báo cáo thành công từ DailyForm
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

// Định dạng ngày hiển thị ngắn gọn
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
    </div>
</template>

<style scoped>
.history-panel {
    display: flex;
    flex-direction: column;
    gap: 20px;
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
    max-height: calc(100vh - 250px);
    overflow-y: auto;
    padding-right: 4px;
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
</style>
