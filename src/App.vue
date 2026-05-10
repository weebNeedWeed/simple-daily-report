<script setup>
import { ref, onMounted, onUnmounted } from "vue";
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

// Global Toast System
const toast = ref({ show: false, message: "", type: "success" });
const showToast = (msg, type = "success") => {
    toast.value = { show: true, message: msg, type };
    setTimeout(() => {
        toast.value.show = false;
    }, 3000);
};

const handleGlobalToast = (e) => {
    const { message, type } = e.detail;
    showToast(message, type);
};

onMounted(() => {
    window.addEventListener("show-toast", handleGlobalToast);
});

onUnmounted(() => {
    window.removeEventListener("show-toast", handleGlobalToast);
});
</script>

<template>
    <div class="app-wrapper">
        <!-- Global Toast Message -->
        <Transition name="fade">
            <div v-if="toast.show" class="toast" :class="toast.type">
                {{ toast.message }}
            </div>
        </Transition>

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
    position: relative;
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

/* Toast Styles */
.toast {
    position: fixed;
    top: 20px;
    right: 20px;
    padding: 12px 24px;
    border-radius: 8px;
    color: white;
    font-weight: 600;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    z-index: 9999;
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
