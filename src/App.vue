<script setup>
import { ref } from "vue";
import ReportHistory from "./components/ReportHistory.vue";
import DailyForm from "./components/DailyForm.vue";

// Lấy ngày hiện tại làm mặc định (YYYY-MM-DD)
const getTodayKey = () => {
    const today = new Date();
    return `${today.getFullYear()}-${String(today.getMonth() + 1).padStart(2, "0")}-${String(today.getDate()).padStart(2, "0")}`;
};

const selectedDate = ref(getTodayKey());

const handleSelectDate = (date) => {
    selectedDate.value = date;
};
</script>

<template>
    <div class="app-wrapper">
        <div class="app-container">
            <aside class="sidebar">
                <ReportHistory
                    @select-date="handleSelectDate"
                    :selected-date="selectedDate"
                />
            </aside>

            <main class="main-content">
                <DailyForm :selected-date="selectedDate" />
            </main>
        </div>
    </div>
</template>

<style>
/* Đảm bảo chiếm trọn màn hình và có nền màu ấm */
.app-wrapper {
    min-height: 100vh;
    background-color: #fffbf1; /* var(--bg-primary) */
}

.app-container {
    display: flex;
    flex-direction: column;
    padding: 20px;
    gap: 40px;
    max-width: 1400px;
    margin: 0 auto;
}

.sidebar {
    width: 100%;
}

.main-content {
    flex: 1;
}

@media (min-width: 1024px) {
    .app-container {
        flex-direction: row;
        padding: 60px 40px;
        align-items: flex-start;
    }

    .sidebar {
        width: 350px;
        position: sticky;
        top: 60px;
    }
}
</style>
