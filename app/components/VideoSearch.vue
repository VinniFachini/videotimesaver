<script setup>
import { ref, computed, watch } from "vue";

const videoUrl = ref("");
const playbackSpeed = ref(1.5);
const duration = ref(null);
const adjustedDuration = ref(null);
const timeSaved = ref(null);
const videoId = ref(null);
let player = null;

// Opções de velocidade disponíveis
const speedOptions = [
    { value: 1, label: "1x (Normal)" },
    { value: 1.25, label: "1.25x" },
    { value: 1.5, label: "1.5x" },
    { value: 1.75, label: "1.75x" },
    { value: 2, label: "2x" },
    { value: 2.25, label: "2.25x" },
    { value: 2.5, label: "2.5x" },
    { value: 2.75, label: "2.75x" },
    { value: 3, label: "3x" },
    { value: 3.25, label: "3.25x" },
    { value: 3.5, label: "3.5x" },
    { value: 3.75, label: "3.75x" },
    { value: 4, label: "4x" }
];

// Função para converter segundos em formato legível
function formatTime(seconds) {
    if (!seconds || isNaN(seconds)) return "0s";

    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = Math.floor(seconds % 60);

    if (hours > 0) {
        return `${hours}h ${minutes}m ${secs}s`;
    } else if (minutes > 0) {
        return `${minutes}m ${secs}s`;
    } else {
        return `${secs}s`;
    }
}

// Calcular tempo economizado
function calculateTimeSavings(originalDuration, speed) {
    const adjusted = originalDuration / speed;
    const saved = originalDuration - adjusted;

    adjustedDuration.value = Math.round(adjusted);
    timeSaved.value = Math.round(saved);

    console.log("Adjusted Duration (seconds):", adjustedDuration.value);
    console.log("Time Saved (seconds):", timeSaved.value);
}

// Extrair VIDEO_ID do link do YouTube
function extractVideoId(url) {
    try {
        const u = new URL(url);
        return u.searchParams.get("v") || u.pathname.split('/').pop();
    } catch {
        // Tentar extrair ID de formato mais simples
        const match = url.match(/(?:youtube\.com\/watch\?v=|youtu\.be\/|youtube\.com\/embed\/)([^&\n?#]+)/);
        return match ? match[1] : null;
    }
}

// Carregar player
function loadPlayer(id, speed) {
    if (player) {
        player.destroy();
        player = null;
    }

    // Resetar valores
    duration.value = null;
    adjustedDuration.value = null;
    timeSaved.value = null;
    videoId.value = id;

    player = new YT.Player("youtube-player", {
        height: "100%",
        width: "100%",
        videoId: id,
        playerVars: {
            'enablejsapi': 1,
            'origin': window.location.protocol + '//' + window.location.hostname + (window.location.port ? ':' + window.location.port : ''),
            'rel': 0,
            'showinfo': 0
        },
        events: {
            onReady: (event) => {
                try {
                    const videoDuration = event.target.getDuration();
                    duration.value = Math.round(videoDuration);
                    event.target.setPlaybackRate(speed);

                    // Calcular economias
                    calculateTimeSavings(videoDuration, speed);

                    console.log("Duration (seconds):", duration.value);
                    console.log("Playback Speed:", speed);

                    // Aplicar border radius
                    setTimeout(() => {
                        const iframe = document.querySelector('#youtube-player iframe');
                        if (iframe) {
                            iframe.style.borderRadius = '0.5rem';
                        }
                    }, 200);
                } catch (error) {
                    console.error("Error in onReady:", error);
                }
            },
            onError: (event) => {
                console.error("YouTube Player Error:", event.data);
                alert("Erro ao carregar o vídeo. Verifique se o URL está correto.");
            }
        }
    });
}

// Carregar API do YouTube
function loadYouTubeAPI() {
    if (window.YT && window.YT.Player) {
        return Promise.resolve();
    }

    return new Promise((resolve) => {
        if (!document.querySelector('script[src*="youtube.com/iframe_api"]')) {
            const tag = document.createElement("script");
            tag.src = "https://www.youtube.com/iframe_api";
            document.head.appendChild(tag);
        }

        window.onYouTubeIframeAPIReady = resolve;
    });
}

// Atualizar velocidade do player em tempo real
function updatePlaybackSpeed() {
    if (player && player.setPlaybackRate) {
        const speed = parseFloat(playbackSpeed.value);
        if (!isNaN(speed) && speed > 0 && speed <= 4) {
            try {
                player.setPlaybackRate(speed);
                // Recalcular economias quando a velocidade mudar
                if (duration.value) {
                    calculateTimeSavings(duration.value, speed);
                }
            } catch (error) {
                console.log("Player not ready yet");
            }
        }
    }
}

// Watch para mudanças na velocidade de reprodução
watch(playbackSpeed, () => {
    updatePlaybackSpeed();
});

// Executar cálculo
async function calculate() {
    const id = extractVideoId(videoUrl.value);
    if (!id) {
        alert("URL do YouTube inválida! Use um formato como: https://www.youtube.com/watch?v=VIDEO_ID");
        return;
    }

    const speed = parseFloat(playbackSpeed.value);
    if (isNaN(speed) || speed <= 0 || speed > 4) {
        alert("Velocidade inválida! Use um valor entre 0.25 e 4.0");
        return;
    }

    try {
        await loadYouTubeAPI();
        loadPlayer(id, speed);
    } catch (error) {
        console.error("Error loading YouTube API:", error);
        alert("Erro ao carregar a API do YouTube.");
    }
}

// Computed para verificar se temos resultados
const hasResults = computed(() => {
    return duration.value !== null && timeSaved.value !== null && adjustedDuration.value !== null;
});
</script>

<template>
    <div class="w-full px-4 sm:px-6 lg:px-8 py-8">
        <div class="max-w-7xl mx-auto">
            <!-- Layout responsivo: coluna única em mobile, duas colunas em desktop -->
            <div class="flex flex-col lg:flex-row lg:items-start gap-6 lg:gap-8">
                
                <!-- Seção Esquerda - Título e Formulário -->
                <div class="w-full lg:w-1/2 order-1 lg:order-1">
                    <!-- Título responsivo -->
                    <h1 class="text-3xl sm:text-4xl lg:text-5xl font-montserrat font-semibold text-slate-800 leading-tight">
                        Video Time Saver
                    </h1>
                    <p class="font-montserrat text-sm sm:text-base lg:text-md text-slate-600 mt-3 lg:mt-2 leading-relaxed">
                        Ever wondered how much time you can save just by speeding up the videos you watch?
                        Here you can see exactly how much! Just enter the video URL and playback speed below!
                    </p>

                    <!-- Formulário -->
                    <div class="mt-6 lg:mt-8">
                        <h2 class="text-lg font-bold mb-4">YouTube Player Info</h2>

                        <!-- Inputs responsivos -->
                        <div class="flex flex-col sm:flex-row sm:items-center gap-4 mb-4">
                            <!-- Input URL -->
                            <div class="flex-1">
                                <input 
                                    v-model="videoUrl" 
                                    type="text" 
                                    placeholder="https://www.youtube.com/watch?v=..." 
                                    class="border px-3 py-2 w-full rounded focus:outline-none focus:ring-2 focus:ring-[#C4302B] text-sm sm:text-base"
                                />
                            </div>
                            
                            <!-- Select Speed -->
                            <div class="">
                                <select 
                                    v-model="playbackSpeed" 
                                    class="border px-3 py-2 w-full sm:w-auto rounded focus:outline-none focus:ring-2 focus:ring-[#C4302B] text-sm sm:text-base"
                                >
                                    <option v-for="option in speedOptions" :key="option.value" :value="option.value">
                                        {{ option.label }}
                                    </option>
                                </select>
                            </div>
                        </div>

                        <!-- Botão Calculate -->
                        <button 
                            @click="calculate" 
                            class="bg-[#C4302B] text-white px-4 py-3 rounded text-base lg:text-lg font-semibold w-full hover:bg-[#e6524c] transition-colors"
                        >
                            Calculate
                        </button>

                        <!-- Resultados dos cálculos -->
                        <div v-if="hasResults" class="mt-6 p-4 bg-gray-50 rounded-lg border">
                            <h3 class="text-base lg:text-lg font-semibold text-slate-800 mb-3">⏱️ Time Savings Results</h3>

                            <div class="space-y-3 text-sm">
                                <div class="flex justify-between items-center">
                                    <span class="text-slate-600">Original Duration:</span>
                                    <span class="font-medium text-slate-800">{{ formatTime(duration) }}</span>
                                </div>
                                <div class="flex justify-between items-center">
                                    <span class="text-slate-600">Adjusted Duration:</span>
                                    <span class="font-medium text-blue-600">{{ formatTime(adjustedDuration) }}</span>
                                </div>
                                <div class="flex justify-between items-center border-t pt-3 mt-3">
                                    <span class="text-slate-600 font-medium">⚡ Time Saved:</span>
                                    <span class="font-bold text-green-600 text-base lg:text-lg">{{ formatTime(timeSaved) }}</span>
                                </div>
                                <div class="flex justify-between items-center">
                                    <span class="text-slate-600">Playback Speed:</span>
                                    <span class="font-medium text-slate-800">{{ playbackSpeed }}x</span>
                                </div>
                            </div>

                            <div class="mt-4 p-3 bg-green-50 rounded border-l-4 border-green-400">
                                <p class="text-sm text-green-700">
                                    🎉 You'll save <strong>{{ formatTime(timeSaved) }}</strong> by watching at <strong>{{ playbackSpeed }}x</strong> speed!
                                </p>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Seção Direita - Player do YouTube -->
                <div class="w-full lg:w-1/2 order-2 lg:order-2">
                    <div class="w-full aspect-video bg-gray-200 rounded-lg overflow-hidden">
                        <div id="youtube-player" class="w-full h-full">
                            <!-- Placeholder quando vídeo não está carregado -->
                            <div v-if="!videoId" class="w-full h-full bg-gray-500 rounded-lg flex items-center justify-center">
                                <div class="text-center text-white p-4">
                                    <div class="text-4xl sm:text-5xl lg:text-6xl mb-4">📺</div>
                                    <p class="text-base lg:text-lg font-semibold mb-2">Video will appear here</p>
                                    <p class="text-sm opacity-75">Enter a YouTube URL and click Calculate</p>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>