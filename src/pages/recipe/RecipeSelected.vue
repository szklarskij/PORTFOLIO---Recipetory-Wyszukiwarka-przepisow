<template>
  <section>
    <div v-if="isLoading">
      <base-spinner></base-spinner>
    </div>
    <base-container v-else-if="!isLoading && !failedFetch">
      <div class="recipe-grid">
        <div class="img-wrapper">
          <img :src="image" :alt="label" @load="onImgLoad" />
        </div>
        <div class="grid-el">
          <div class="label">
            <div class="label-div">
              <h2>
                {{ label }}
              </h2>
              <p class="bookmark" v-if="!bookmarked" @click="addBookmark">
                <ion-icon name="bookmark-outline"></ion-icon>
              </p>
              <p class="bookmark" v-else @click="removeBookmark">
                <ion-icon name="bookmark"></ion-icon>
              </p>
            </div>
            <p>By {{ source }}</p>
          </div>

          <div v-if="badges">
            <base-badge
              v-for="badge in badges"
              :key="badge"
              :type="badge"
            ></base-badge>
          </div>
          <div class="nutrients">
            <ul class="nutrients-list">
              <li class="nutrients-el">
                Energy: {{ totalNutrients.ENERC_KCAL.quantity.toFixed(0) }}
                {{ totalNutrients.ENERC_KCAL.unit }}
              </li>
              <li class="nutrients-el">
                Protein: {{ totalNutrients.PROCNT.quantity.toFixed(0) }}
                {{ totalNutrients.PROCNT.unit }}
              </li>
              <li class="nutrients-el">
                Fat: {{ totalNutrients.FAT.quantity.toFixed(0) }}
                {{ totalNutrients.FAT.unit }}
              </li>
              <li class="nutrients-el">
                Carbs: {{ totalNutrients.CHOCDF.quantity.toFixed(0) }}
                {{ totalNutrients.CHOCDF.unit }}
              </li>
              <li class="nutrients-el">
                Cholesterol: {{ totalNutrients.CHOLE.quantity.toFixed(0) }}
                {{ totalNutrients.CHOLE.unit }}
              </li>
              <li class="nutrients-el">
                Sugars: {{ totalNutrients.SUGAR.quantity.toFixed(0) }}
                {{ totalNutrients.SUGAR.unit }}
              </li>
              <li class="nutrients-el">
                Total weight: {{ totalWeight.toFixed(0) }} g
              </li>
            </ul>
            <p class="title" v-if="time !== 0">
              Preparation time: {{ time }} min
            </p>
          </div>
        </div>
      </div>
      <div class="ingredients-container">
        <p class="title">
          Recipe ingredients per<input
            type="number"
            min="1"
            max="20"
            v-model="servingsInput"
          />
          servings
        </p>

        <div class="ingredients">
          <ul class="igredients-list">
            <li
              class="igredients-el"
              v-for="ing in ingredients"
              :key="ing.text"
            >
              <ion-icon name="checkmark-outline"></ion-icon>
              {{
                ing.quantity === 0
                  ? ""
                  : Math.round(
                      ((ing.quantity * servingsInput) / servings) * 100
                    ) / 100
              }}
              {{
                ing.measure
                  ? ing.measure.includes("unit")
                    ? ""
                    : ing.measure
                  : ""
              }}
              {{ ing.unit }}
              {{ ing.food }}
            </li>
          </ul>
          <div></div>
        </div>

        <div class="link-to-recipe">
          <base-button url :urlto="url">Go to the recipe's page!</base-button>
        </div>
      </div>
    </base-container>
    <base-container v-else>
      <h2>Something goes wrong!</h2>
      <p>Check your internet connnection and try again.</p>
    </base-container>
  </section>
</template>
<script>
import { ref, computed, watch } from "vue";
import { useStore } from "vuex";
import { useRoute, useRouter } from "vue-router";
import { EDAMAM_ID } from "../../config.js";
import { EDAMAM_KEY } from "../../config.js";
import { LABELS_ARR } from "../../config.js";
export default {
  setup() {
    const store = useStore();
    const route = useRoute();
    const router = useRouter();
    const onImgLoad = function () {
      console.log("img loaded");
      imageLoaded.value = true;
    };
    const bookmarked = ref(false);
    const imageLoaded = ref(false);
    const failedFetch = ref(false);
    let thisRecipe = null;

    const recipes = store.getters["search/getSearchList"];
    const isLoading = ref(true);

    ///////////////////////////////////////////////////////////////////////recipe elements
    const label = computed(function () {
      return thisRecipe.label;
    });
    const source = computed(function () {
      return thisRecipe.source;
    });
    const image = computed(function () {
      return thisRecipe.imageLarge;
    });
    const badges = computed(function () {
      return thisRecipe.healthLabels;
    });
    const calories = computed(function () {
      return Math.floor(thisRecipe.calories);
    });
    const servings = computed(function () {
      return thisRecipe.servings;
    });

    const servingsInput = ref(null);

    const time = computed(function () {
      return thisRecipe.totalTime;
    });
    const totalWeight = computed(function () {
      return thisRecipe.totalWeight;
    });
    const url = computed(function () {
      return thisRecipe.url;
    });
    const ingredients = computed(function () {
      return thisRecipe.ingredients;
    });
    const healthLabels = computed(function () {
      return thisRecipe.healthLabels;
    });
    const totalNutrients = computed(function () {
      return thisRecipe.totalNutrients;
    });
    /////////////////////////////////////////////////////////////////////// bookmarks
    const addBookmark = function () {
      bookmarked.value = true;
    };
    const removeBookmark = function () {
      bookmarked.value = false;
    };

    /////////////////////////////////////////////////////////////////////// validate input

    watch(servingsInput, (newVal, oldVal) => {
      if (newVal <= 0 || newVal > 20) servingsInput.value = oldVal;
    });

    /////////////////////////////////////////////////////////////////////// init
    const init = async function () {
      if (recipes.length !== 0) {
        // console.log(recipes);

        thisRecipe = recipes.find((recipe) => recipe.id === route.params.id);
        loadImage();
      } else if (store.getters["recipe/getRecipe"]) {
        thisRecipe = store.getters["recipe/getRecipe"];
        loadImage();
      } else {
        try {
          const response = await fetch(
            `https://api.edamam.com/api/recipes/v2/${route.params.id}?type=public&app_id=${EDAMAM_ID}&app_key=${EDAMAM_KEY}`
          );

          ////label arr do configa
          let filteredLabels = [];

          const data = await response.json();
          if (!response.ok) {
            let error;
            if (response.status === 404) {
              error = new Error("Wrong recipe id at url.");
            } else {
              error = new Error("Failed to fetch data from server.");
            }

            router.replace("/search");
            throw error;
          } else {
            LABELS_ARR.forEach((label) => {
              if (data.recipe.healthLabels.includes(label)) {
                filteredLabels.push(label.toLowerCase());
              }
            });
            thisRecipe = {
              id: route.params.id,
              label: data.recipe.label,
              image: data.recipe.image,
              imageLarge: data.recipe.images.LARGE
                ? data.recipe.images.LARGE.url
                : data.recipe.image,
              source: data.recipe.source,
              calories: data.recipe.calories,
              healthLabels: filteredLabels,
              url: data.recipe.url,
              ingredients: data.recipe.ingredients,
              totalNutrients: data.recipe.totalNutrients,
              totalTime: data.recipe.totalTime,
              totalWeight: data.recipe.totalWeight,
              servings: data.recipe.yield,
            };

            store.dispatch("recipe/setRecipe", thisRecipe);
            loadImage();
          }
        } catch (error) {
          isLoading.value = false;
          failedFetch.value = true;
          store.dispatch("search/setError", error);
        }
        console.log(thisRecipe);
      }
    };
    const loadImage = function () {
      try {
        servingsInput.value = thisRecipe.servings;
        const img = new Image();
        img.src = thisRecipe.imageLarge;
        img.onload = function () {
          // store.dispatch('search/spinnerOff')
          isLoading.value = false;
        };
      } catch (error) {
        store.dispatch("search/setError", error);
      }
    };

    init();
    return {
      isLoading,
      onImgLoad,
      imageLoaded,
      label,
      source,
      image,
      badges,
      calories,
      servings,
      time,
      totalWeight,
      url,
      ingredients,
      healthLabels,
      totalNutrients,
      servingsInput,
      failedFetch,
      bookmarked,
      addBookmark,
      removeBookmark,
    };
  },
};
</script>
{
<style scoped>
h2 {
  text-transform: uppercase;
  padding-top: 1rem;
  font-size: 1.8rem;
  margin: 0;
}
.label {
  /* display: inline; */
  margin: 0;
  padding-bottom: 1rem;
}
.label-div {
  display: flex;
  justify-content: space-between;
  align-content: center;
}

.title {
  display: flex;
  font-size: 1.3rem;
  align-self: center;
  /* text-align: center; */
}
.bookmark {
  font-size: 4rem;
  padding: 0;
}
.bookmark:hover,
.bookmark:active {
  cursor: pointer;
}
.bookmark ion-icon {
  font-size: 2rem;
  padding: 1rem 1rem 0 1rem;
  /* color: rgb(0, 0, 0); */
}
input {
  font-size: 1.3rem;
  width: 3rem;
  margin-left: 0.5rem;
  margin-right: 0.5rem;
}

.label p {
  font-size: 1.2rem;
  margin: 0;
  /* padding-bottom: 1rem; */
}

img {
  object-fit: fill;
  transition: 0.6s ease;
  height: 100%;
  width: 100%;
}
.img-wrapper {
  border-top-left-radius: 12px;
  display: inline-block;
  overflow: hidden;
  height: 100%;
  width: auto;
  -webkit-transition: 0.6s ease;
  transition: 0.6s ease;
  max-width: 100%;
}

img:hover {
  -webkit-transition: 0.6s ease;
  transition: 0.6s ease;
  transform: scale(1.5);
  -ms-transform: scale(1.5); /* IE 9 */
  -moz-transform: scale(1.5); /* Firefox */
  -webkit-transform: scale(1.5); /* Safari and Chrome */
  -o-transform: scale(1.5); /* Opera */
}

.recipe-grid {
  display: grid;
  gap: 1rem;
  grid-template-columns: 1fr 1.5fr;
  grid-template-rows: 1fr;
  border-top-right-radius: 12px;
  border-top-left-radius: 12px;
  background-color: rgb(231, 231, 231);
  box-shadow: 0 2px 2px 0 rgba(0, 0, 0, 0.26);
}

.ingredients-container {
  display: flex;
  flex-direction: column;
  padding: 2rem 8rem 2rem 8rem;
}

.ingredients {
  margin: 2rem 5rem 3rem 5rem;
}

.nutrients-list {
  list-style: none;
  padding: 0;
}
.nutrients-el {
  margin: 0;
  margin-bottom: 0.5rem;
  font-size: 1rem;
}

.igredients-list {
  gap: 1rem;
  display: grid;
  grid-template-columns: 1fr 1fr;
  list-style: none;
  padding: 0;
  margin: 0;
}
.igredients-el {
  margin: 0;
  font-size: 1rem;
}
.link-to-recipe {
  align-self: center;
}
</style>
}