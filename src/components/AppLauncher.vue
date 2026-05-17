<script setup>
import { ref, onMounted, onUnmounted, watch, nextTick } from "vue";

const isOpen = ref(false);
const searchQuery = ref("");
const results = ref([]);
const selectedIndex = ref(0);
const inputRef = ref(null);

const toggleLauncher = (e) => {
    // Ngăn chặn Ctrl+K mặc định của trình duyệt (focus search bar)
    if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === "k") {
        e.preventDefault();
        isOpen.value = !isOpen.value;
        if (isOpen.value) {
            searchQuery.value = "";
            results.value = [];
            selectedIndex.value = 0;
            nextTick(() => {
                inputRef.value?.focus();
            });
        }
    }
    if (e.key === "Escape" && isOpen.value) {
        isOpen.value = false;
    }
};

const handleSearch = () => {
    const query = searchQuery.value.trim().toLowerCase();
    results.value = [];
    selectedIndex.value = 0;

    if (!query) return;

    // Lệnh thao tác nhanh
    if (query.startsWith("/")) {
        const commands = [
            {
                id: "cmd_today",
                title: "Đi đến Hôm nay",
                type: "command",
                action: "today",
            },
            {
                id: "cmd_export",
                title: "Xuất dữ liệu (Backup)",
                type: "command",
                action: "export",
            },
        ];
        const term = query.substring(1);
        results.value = commands.filter(
            (cmd) =>
                cmd.title.toLowerCase().includes(term) ||
                cmd.action.includes(term),
        );
        return;
    }

    // Điều hướng ngày siêu tốc (DD/MM)
    const dateMatch = query.match(/^(\d{1,2})\/(\d{1,2})$/);
    if (dateMatch) {
        const day = dateMatch[1].padStart(2, "0");
        const month = dateMatch[2].padStart(2, "0");
        const year = new Date().getFullYear();
        const dateKey = `${year}-${month}-${day}`;
        results.value.push({
            id: `date_${dateKey}`,
            title: `Đi đến báo cáo ngày ${day}/${month}/${year}`,
            type: "date",
            dateKey: dateKey,
        });
        return;
    }

    // Tìm kiếm toàn cục
    const history = JSON.parse(localStorage.getItem("dr_history") || "[]");
    const found = [];

    for (const dateKey of history) {
        const dataStr = localStorage.getItem(`dr_data_${dateKey}`);
        if (!dataStr) continue;

        try {
            const data = JSON.parse(dataStr);
            let matchText = "";

            // Tìm trong các phần chính
            ["completed", "next", "blockers"].forEach((field) => {
                if (data[field] && data[field].toLowerCase().includes(query)) {
                    matchText = extractSnippet(data[field], query);
                }
            });

            // Tìm trong ghi chú con
            if (!matchText && data.notes && data.notes.length > 0) {
                for (const note of data.notes) {
                    if (
                        (note.title &&
                            note.title.toLowerCase().includes(query)) ||
                        (note.content &&
                            note.content.toLowerCase().includes(query))
                    ) {
                        matchText = extractSnippet(
                            `${note.title} ${note.content}`,
                            query,
                        );
                        break;
                    }
                }
            }

            if (matchText) {
                found.push({
                    id: dateKey,
                    title: formatDate(dateKey),
                    snippet: matchText,
                    type: "report",
                    dateKey: dateKey,
                });
            }
        } catch (e) {
            // Bỏ qua lỗi parse
        }
    }

    results.value = found;
};

const extractSnippet = (text, query) => {
    const idx = text.toLowerCase().indexOf(query);
    if (idx === -1) return "";
    const start = Math.max(0, idx - 20);
    const end = Math.min(text.length, idx + query.length + 20);
    let snippet = text.substring(start, end).replace(/\n/g, " ");
    if (start > 0) snippet = "..." + snippet;
    if (end < text.length) snippet = snippet + "...";
    return snippet;
};

const formatDate = (dateStr) => {
    const date = new Date(dateStr);
    return date.toLocaleDateString("vi-VN", {
        day: "2-digit",
        month: "2-digit",
        year: "numeric",
    });
};

watch(searchQuery, handleSearch);

const handleKeyDown = (e) => {
    if (!isOpen.value) return;

    if (e.key === "ArrowDown") {
        e.preventDefault();
        if (selectedIndex.value < results.value.length - 1) {
            selectedIndex.value++;
            scrollToSelected();
        }
    } else if (e.key === "ArrowUp") {
        e.preventDefault();
        if (selectedIndex.value > 0) {
            selectedIndex.value--;
            scrollToSelected();
        }
    } else if (e.key === "Enter") {
        e.preventDefault();
        if (results.value.length > 0) {
            selectResult(results.value[selectedIndex.value]);
        }
    }
};

const scrollToSelected = () => {
    nextTick(() => {
        const selectedEl = document.querySelector(".result-item.is-selected");
        if (selectedEl) {
            selectedEl.scrollIntoView({ block: "nearest", behavior: "smooth" });
        }
    });
};

const selectResult = (result) => {
    if (result.type === "command") {
        if (result.action === "today") {
            const today = new Date();
            const todayKey = `${today.getFullYear()}-${String(today.getMonth() + 1).padStart(2, "0")}-${String(today.getDate()).padStart(2, "0")}`;
            window.dispatchEvent(
                new CustomEvent("change-date", { detail: todayKey }),
            );
        } else if (result.action === "export") {
            window.dispatchEvent(new CustomEvent("trigger-export"));
        }
    } else if (result.type === "date") {
        const history = JSON.parse(localStorage.getItem("dr_history") || "[]");
        if (history.includes(result.dateKey)) {
            window.dispatchEvent(
                new CustomEvent("change-date", { detail: result.dateKey }),
            );
        } else {
            window.dispatchEvent(
                new CustomEvent("request-new-date", { detail: result.dateKey }),
            );
        }
    } else if (result.type === "report") {
        window.dispatchEvent(
            new CustomEvent("change-date", { detail: result.dateKey }),
        );
    }

    isOpen.value = false;
};

const openLauncher = () => {
    isOpen.value = true;
    searchQuery.value = "";
    results.value = [];
    selectedIndex.value = 0;
    nextTick(() => {
        inputRef.value?.focus();
    });
};

onMounted(() => {
    window.addEventListener("keydown", toggleLauncher);
    window.addEventListener("open-launcher", openLauncher);
});

onUnmounted(() => {
    window.removeEventListener("keydown", toggleLauncher);
    window.removeEventListener("open-launcher", openLauncher);
});
</script>

<template>
    <Teleport to="body">
        <div
            v-if="isOpen"
            class="launcher-overlay"
            @click.self="isOpen = false"
        >
            <div class="launcher-box">
                <div class="launcher-header">
                    <input
                        ref="inputRef"
                        type="text"
                        v-model="searchQuery"
                        placeholder="Tìm báo cáo hoặc nhập lệnh (/, DD/MM)..."
                        @keydown="handleKeyDown"
                    />
                </div>
                <ul v-if="results.length > 0" class="launcher-results">
                    <li
                        v-for="(result, index) in results"
                        :key="result.id"
                        class="result-item"
                        :class="{ 'is-selected': selectedIndex === index }"
                        @click="selectResult(result)"
                        @mouseenter="selectedIndex = index"
                    >
                        <div class="result-title">{{ result.title }}</div>
                        <div v-if="result.snippet" class="result-snippet">
                            {{ result.snippet }}
                        </div>
                    </li>
                </ul>
                <div v-else-if="searchQuery" class="launcher-empty">
                    Không tìm thấy kết quả phù hợp.
                </div>
                <div v-else class="launcher-help">
                    <p>
                        Mẹo: Gõ <strong>/today</strong> để về hôm nay,
                        <strong>/export</strong> để sao lưu, hoặc ngày
                        <strong>DD/MM</strong>.
                    </p>
                </div>
            </div>
        </div>
    </Teleport>
</template>

<style scoped>
.launcher-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.4);
    backdrop-filter: blur(4px);
    z-index: 4000;
    display: flex;
    justify-content: center;
    align-items: flex-start;
    padding-top: 15vh;
}

.launcher-box {
    background-color: #ffffff;
    border-radius: 12px;
    width: 90%;
    max-width: 600px;
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.2);
    border: 1px solid var(--border-color);
    display: flex;
    flex-direction: column;
    overflow: hidden;
    animation: slideDown 0.2s ease-out;
}

.launcher-header input {
    width: 100%;
    border: none;
    border-bottom: 1px solid var(--border-color);
    padding: 20px;
    font-size: 1.2rem;
    background: transparent;
    outline: none;
    color: var(--text-dark);
}

.launcher-header input:focus {
    border-bottom-color: var(--color-primary);
}

.launcher-results {
    list-style: none;
    margin: 0;
    padding: 10px;
    max-height: 400px;
    overflow-y: auto;
}

.result-item {
    padding: 12px 16px;
    border-radius: 8px;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    gap: 4px;
    transition: var(--transition-fast);
}

.result-item.is-selected {
    background-color: var(--bg-secondary);
    color: var(--color-primary);
}

.result-item.is-selected .result-title {
    color: var(--color-primary);
}

.result-title {
    font-weight: 700;
    font-size: 1.05rem;
    color: var(--text-dark);
}

.result-snippet {
    font-size: 0.85rem;
    color: var(--text-medium);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.launcher-empty,
.launcher-help {
    padding: 24px;
    text-align: center;
    color: var(--text-medium);
    font-size: 0.95rem;
}

@keyframes slideDown {
    from {
        opacity: 0;
        transform: translateY(-20px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}
</style>
