<script setup>
import { ref, onMounted } from 'vue';
import { getPokemonLink, getPokemonDetail, getPokemonImage, getPokemonEvolution } from '@/requests/index';
import { getPokemonId, formatPokemonId, getChineseNames, getPokemonEvolutionUrls } from '@/utils/utils'
import { Search } from '@element-plus/icons-vue'

const limit = 20;
const offset = ref(0);
const pokemons = ref([]);
let totalResults = 0;
const lastOffset = ref(0);
const displayedPokemons = ref([]);
const isModalVisible = ref(false);
const selectedPokemon = ref(null);

onMounted(() => {
    fetchData(offset.value);
    preloadNextPage();
});

const fetchData = (offset) => {
    getPokemonLink(offset, limit)
        .then(async ({ data }) => {
            totalResults = data.count;
            lastOffset.value = Math.floor((totalResults - 1) / limit) * limit;
            // Get Chinese names for each Pokemon
            const pokemonIds = data.results.map(pokemon => getPokemonId(pokemon.url));
            const chineseNames = await getChineseNames(pokemonIds);

            pokemons.value = data.results.map((pokemon, index) => ({
                name: chineseNames[index].name, // Use Chinese name
                chainUrl: chineseNames[index].chainUrl,
                varieties: chineseNames[index].varieties,
                constid: formatPokemonId(getPokemonId(pokemon.url)),
                id: getPokemonId(pokemon.url),
                image: getPokemonImage(getPokemonId(pokemon.url)),
                valid: true
            }));
            console.log('Pokemons:', pokemons.value); // Add this line for debugging

            displayedPokemons.value = pokemons.value;
            console.log('Displayed Pokemons:', displayedPokemons.value);
        })
        .catch(error => {
            console.error('Error fetching Pokemon data:', error);
        });
};

const preloadNextPage = () => {
    const nextOffset = offset.value + limit * 10;
    if (nextOffset <= lastOffset.value) {
        getPokemonLink(nextOffset, limit)
            .then(({ data }) => {
                // Preload images for the next page
                data.results.forEach(pokemon => {
                    const imageUrl = getPokemonImage(pokemon.url);
                    const image = new Image();
                    image.src = imageUrl;
                });
            })
            .catch(error => {
                console.error('Error preloading next page:', error);
            });
    }
};

const prevPage = () => {
    if (offset.value >= limit) {
        offset.value -= limit;
        // console.log('Prev page offset:', offset.value); // Add this line for debugging
        fetchData(offset.value);
        preloadNextPage();
    }
};

const nextPage = () => {
    if (lastOffset.value > offset.value) {
        offset.value += limit;
        // console.log('Next page offset:', offset.value); // Add this line for debugging
        fetchData(offset.value);
        preloadNextPage();
    }
};

const openDetailModal = (pokemon) => {
    console.log('Selected Pokemon:', pokemon);
    selectedPokemon.value = {
        ...pokemon,
    }
    isModalVisible.value = true;

    // 获取宝可梦的详细信息
    getPokemonDetail(pokemon.id)
        .then(({ data }) => {
            selectedPokemon.value = {
                ...selectedPokemon.value,
                height: data.height,
                weight: data.weight,
            };
        })
    getPokemonEvolution(pokemon.chainUrl)
        .then((chainResponse) => {
            const evolutionUrls = getPokemonEvolutionUrls(chainResponse.data.chain);
            console.log('Evolution URLs:', evolutionUrls);
            selectedPokemon.value = {
                ...selectedPokemon.value,
                evolutionUrls: evolutionUrls// 将进化链 URL 数组存储在 selectedPokemon 中
            };
        })

};

const handleEvolutionClick = (id) => {
    // 处理点击事件的逻辑，可以在此处实现你想要的行为
    // console.log('Clicked on evolution image:', id);
    handleSinglePokemon(id);
};

const handleSinglePokemon = (id) => {
    getChineseNames([id]) // 传递数组给 getChineseNames，因为它期望一个 ID 数组
        .then((chineseNames) => {
            const chineseName = chineseNames[0]; // 获取数组中的第一个元素，即对应于传入 ID 的中文名称
            getPokemonDetail(id)
                .then(() => {
                    const formattedPokemon = {
                        name: chineseName.name,
                        chainUrl: chineseName.chainUrl,
                        varieties: chineseName.varieties,
                        constid: formatPokemonId(id),
                        id: id,
                        image: getPokemonImage(id),
                        valid: true
                    };
                    // 调用 openDetailModal 函数
                    openDetailModal(formattedPokemon);
                })
                .catch(error => {
                    console.error('Error fetching Pokemon detail:', error);
                });
        })
        .catch(error => {
            console.error('Error fetching Chinese names:', error);
        });
};

const searchId = ref('')
const searchPokemon = (searchId) => {
    // 首先检查搜索的 ID 是否在范围内
    if (searchId.value < 1 || searchId.value > totalResults) {
        throw new Error('Invalid Pokemon ID');
    } else {
        // 如果搜索的 ID 在范围内，执行 handleSinglePokemon 方法
        handleSinglePokemon(searchId);
    }
};

const closeDetailModal = () => {
    isModalVisible.value = false;
};

const handlePageChange = (page) => {
    offset.value = (page - 1) * limit;
    fetchData(offset.value);
};

</script>

<template>
    <div>
        <h1>寶可夢圖鑑</h1>
        <div>
            <button @click="prevPage" :disabled="offset <= 0">上一頁</button>
            <button @click="nextPage" :disabled="offset >= lastOffset">下一頁</button>
            <label style="width: 180px">
                <input type="text" v-model="searchId" placeholder="输入要搜索的ID">
                <el-button type="primary" value="Submit" size="small" :icon="Search"
                    @click="searchPokemon(searchId)">Search</el-button>
            </label>
            <el-pagination background layout="prev, pager, next" :total="1302" :page-size="20"
                @current-change="handlePageChange" :disabled="offset < 0 || offset > lastOffset" />
        </div>
        <div class="pokemon-container" v-if="displayedPokemons.length > 0">
            <div v-for="(pokemon, index) in displayedPokemons" :key="index" class="pokemon-item">
                <div class="pokemon-image" @click="openDetailModal(pokemon)">
                    <img v-if="pokemon.image" :src="pokemon.image" :alt="pokemon.name" />
                    <div v-if="pokemon.image" class="pokemon-info">
                        <div class="pokemon-id">{{ pokemon.constid }}</div>
                        <div class="pokemon-name">{{ pokemon.name }}</div>
                    </div>
                </div>
            </div>
        </div>
        <!-- Modal -->
        <el-dialog v-model="isModalVisible" :show-close="false" width="500">
            <div class="dialog-header">
                <h1>Pokemon Detail</h1>
                <el-button @click="closeDetailModal">Close</el-button>
            </div>
            <div v-if="selectedPokemon" class="dialog-container">
                <p>編號：{{ selectedPokemon.constid }}</p>
                <p>名字：{{ selectedPokemon.name }}</p>
                <img class="dialog-container-img" :src="selectedPokemon.image" :alt="selectedPokemon.id" />
                <p>身高：{{ selectedPokemon.height / 10 || 0 }}m</p>
                <p>體重：{{ selectedPokemon.weight / 10 || 0 }}kg</p>
                <!-- 其他寶可夢資訊 -->
                <div class="evolution-body" v-if="selectedPokemon.evolutionUrls">
                    <h2>進化鏈</h2>
                    <div v-if="selectedPokemon.evolutionUrls.length > 1" class="evolution-container">
                        <div v-for="(url, index) in selectedPokemon.evolutionUrls" :key="index" class="evolution-item">
                            <img class="evolutionImg" :src="url.evolutionUrls" alt="Evolution Image"
                                @click="handleEvolutionClick(url.speciesId)" />
                            <div>{{ url.name }}</div>
                        </div>
                    </div>
                    <div v-else>
                        不能進化
                    </div>
                </div>
                <div class="variety-body" v-if="selectedPokemon.varieties">
                    <h2>寶可夢其他樣子</h2>
                    <div v-if="selectedPokemon.varieties.length > 1" class="variety-container">
                        <div v-for="(variety, index) in selectedPokemon.varieties" :key="index">
                            <img class="varietyImg" :src="variety.url" alt="Pokemon Image"
                                @click="handleEvolutionClick(variety.id)" />
                            <div>{{ variety.name }}</div>
                        </div>
                    </div>
                    <div v-else>
                        沒有其他型態
                    </div>
                </div>
            </div>
        </el-dialog>
    </div>
</template>

<style>
@media screen and (max-width: 1024px) {
    .pokemon-container {
        min-width: 1024px;
    }
}

/* 修改 CSS 样式 */
.pokemon-container {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    background-image: url('https://tw.portal-pokemon.com/play/resources/pokedex/img/list_bg.jpg');
    background-size: cover;
    background-repeat: no-repeat;
    background-position: center;
    /* 自動調整高度 */
}

.pokemon-item {
    width: calc(25% - 10px);
    display: flex;
    align-items: center;
    padding: 5px;
    margin-bottom: 5px;
    /* 調整每個格子之間的間距 */
    height: auto;
    /* 設置長方形的高度 */
}

.pokemon-image {
    width: 100%;
    /* 讓每個圖片填滿父容器的寬度 */
    height: auto;
    /* 設置圖片高度自動調整 */
    min-width: 228px;
    /* 設置圖片最小寬度為 228px */
    max-width: 230px;
    /* 設置圖片最大寬度為 287px */
    min-height: 350px;
    /* 設置圖片最小高度為 341px */
    max-height: 350px;
    /* 設置圖片最大高度為 438px */
    position: relative;
    background-image: url('https://tw.portal-pokemon.com/play/resources/pokedex/img/list_pokemon_bg.png');
    background-position: center;
    background-repeat: no-repeat;
    background-size: cover;
    /* 使用 cover 讓背景圖填滿整個區域 */
}

img {
    position: absolute;
    width: 60%;
    left: 20%;
    top: 8%;
}

.evolution-container,
.variety-container {
    display: flex;
}

.evolutionImg {
    position: relative;
}

.varietyImg {
    position: relative;
}

.pokemon-info {
    margin-top: 5px;
    text-align: center;
}

.pokemon-id {
    font-size: 20px;
    margin: 5px 0;
    position: absolute;
    left: 10%;
    top: 57%;
    color: #7bf2f0;
}

.pokemon-name {
    font-size: 20px;
    margin: 5px 0;
    padding: 5px 0;
    position: absolute;
    left: 10%;
    top: 62%;
    color: #fff;
}

.dialog-header {
    display: flex;
    justify-content: space-between;
}

.dialog-container,
.dialog-container-img {
    position: relative;
}
</style>
