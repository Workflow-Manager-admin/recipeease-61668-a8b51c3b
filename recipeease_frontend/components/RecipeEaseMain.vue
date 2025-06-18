<template>
  <div class="recipeease-container" :style="rootStyle">
    <aside class="sidebar" :style="sidebarStyle">
      <div class="logo">
        <span role="img" aria-label="logo">🍽️</span>
        <span class="logo-title">RecipeEase</span>
      </div>
      <nav class="category-list">
        <button
          v-for="cat in categories"
          :key="cat"
          :class="['category-btn', { active: cat === selectedCategory }]"
          @click="selectCategory(cat)"
        >
          {{ cat }}
        </button>
      </nav>
      <div class="favorites" v-if="isAuthenticated">
        <h4>Favorites</h4>
        <ul>
          <li v-for="fav in favoriteRecipes" :key="fav.id">{{ fav.name }}</li>
        </ul>
      </div>
      <div class="sidebar-auth">
        <button v-if="!isAuthenticated" @click="showAuthModal = true" class="auth-btn">
          Login / Register
        </button>
        <div v-else class="user-profile">
          <span class="user-icon">👤</span>
          <span>{{ currentUser.username }}</span>
          <button class="logout-btn" @click="logout">Logout</button>
        </div>
      </div>
    </aside>
    <main class="main-content" :style="mainStyle">
      <header class="recipeease-header">
        <input
          class="search-bar"
          type="search"
          v-model.trim="searchQuery"
          placeholder="Search recipes by name, ingredient, or category"
          @input="search"
        />
      </header>
      <!-- Main area: list or recipe detail depending on state -->
      <div v-if="showDetail">
        <section class="recipe-detail">
          <button class="back-btn" @click="showDetail = false">← Back</button>
          <h2>{{ recipeDetail.name }}</h2>
          <div class="detail-meta">
            <span class="category">{{ recipeDetail.category }}</span>
            <span class="time">{{ recipeDetail.cookTime }} min</span>
            <button
              v-if="isAuthenticated"
              @click="toggleFavorite(recipeDetail.id)"
              :class="['favorite-btn', { filled: isFavorite(recipeDetail.id) }]"
            >
              <span v-if="isFavorite(recipeDetail.id)">★</span>
              <span v-else>☆</span>
              Favorite
            </button>
          </div>
          <h3>Ingredients</h3>
          <ul>
            <li v-for="item in recipeDetail.ingredients" :key="item">{{ item }}</li>
          </ul>
          <h3>Instructions</h3>
          <ol>
            <li v-for="step in recipeDetail.instructions" :key="step.slice(0,20)">
              {{ step }}
            </li>
          </ol>
        </section>
      </div>
      <div v-else>
        <!-- Recipe browsing and filtering -->
        <section class="recipe-list">
          <div class="recipes-grid">
            <article
              v-for="recipe in filteredRecipes"
              :key="recipe.id"
              class="recipe-card"
              @click="openRecipe(recipe)"
              tabindex="0"
            >
              <img :src="recipe.image" :alt="recipe.name" class="recipe-img" />
              <h4>{{ recipe.name }}</h4>
              <div class="meta">
                <span class="category">{{ recipe.category }}</span>
                <span class="time">{{ recipe.cookTime }} min</span>
              </div>
              <button
                v-if="isAuthenticated"
                @click.stop="toggleFavorite(recipe.id)"
                :class="['favorite-btn', { filled: isFavorite(recipe.id) }]"
                :title="isFavorite(recipe.id) ? 'Remove from favorites': 'Add to favorites'"
              >
                <span v-if="isFavorite(recipe.id)">★</span>
                <span v-else>☆</span>
              </button>
            </article>
            <div v-if="filteredRecipes.length === 0" class="no-results">
              <p>No recipes found. Try adjusting your search or category filter.</p>
            </div>
          </div>
        </section>
      </div>
      <!-- Modal for authentication -->
      <div class="modal-backdrop" v-if="showAuthModal" @click.self="showAuthModal = false">
        <div class="auth-modal">
          <button class="modal-close" @click="showAuthModal = false">✕</button>
          <h3 v-if="!isRegisterMode">Login</h3>
          <h3 v-else>Register</h3>
          <form @submit.prevent="isRegisterMode ? register() : login()">
            <input v-model.trim="authForm.username" type="text" placeholder="Username" required />
            <input v-model.trim="authForm.password" type="password" placeholder="Password" required minlength="4" />
            <button type="submit" class="modal-action-btn">
              {{ isRegisterMode ? "Register" : "Login" }}
            </button>
          </form>
          <div class="switch-auth-mode">
            <button @click="isRegisterMode = !isRegisterMode" class="switch-btn">
              {{ isRegisterMode ? "Already have an account? Login" : "Don't have an account? Register" }}
            </button>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
// PUBLIC_INTERFACE
export default {
  name: "RecipeEaseMain",
  data() {
    return {
      // App state
      categories: [
        "All",
        "Breakfast",
        "Lunch",
        "Dinner",
        "Dessert",
        "Snack",
        "Vegetarian",
        "Vegan",
        "Gluten-free",
      ],
      selectedCategory: "All",
      searchQuery: "",
      isAuthenticated: false,
      currentUser: null,
      showAuthModal: false,
      isRegisterMode: false,
      authForm: {
        username: "",
        password: "",
      },

      showDetail: false,
      recipeDetail: null,

      // Simulated recipes and users -- In real world, use API calls
      recipes: [
        {
          id: 1,
          name: "Classic Pancakes",
          image: "https://images.unsplash.com/photo-1504674900247-0877df9cc836?w=400&q=80",
          category: "Breakfast",
          cookTime: 20,
          ingredients: ["Flour", "Eggs", "Milk", "Sugar", "Baking Powder", "Salt"],
          instructions: [
            "Mix dry ingredients together.",
            "Whisk in eggs and milk.",
            "Cook on skillet until golden brown.",
          ],
        },
        {
          id: 2,
          name: "Vegetarian Chili",
          image: "https://images.unsplash.com/photo-1464305795204-6f5bbfc7fb81?w=400&q=80",
          category: "Vegetarian",
          cookTime: 45,
          ingredients: [
            "Beans",
            "Tomatoes",
            "Bell Peppers",
            "Corn",
            "Chili Powder",
            "Onions",
          ],
          instructions: [
            "Sauté onions and peppers.",
            "Add beans and tomatoes.",
            "Simmer with spices.",
          ],
        },
        {
          id: 3,
          name: "Chocolate Cake",
          image: "https://images.unsplash.com/photo-1519864600265-abb23847ef2c?w=400&q=80",
          category: "Dessert",
          cookTime: 60,
          ingredients: [
            "Cocoa Powder",
            "Flour",
            "Eggs",
            "Butter",
            "Sugar",
            "Baking Soda",
          ],
          instructions: [
            "Mix dry and wet ingredients separately.",
            "Combine and pour into pan.",
            "Bake until done.",
          ],
        },
        {
          id: 4,
          name: "Grilled Chicken Salad",
          image: "https://images.unsplash.com/photo-1512621776951-a57141f2eefd?w=400&q=80",
          category: "Lunch",
          cookTime: 30,
          ingredients: [
            "Chicken Breast",
            "Lettuce",
            "Tomatoes",
            "Cucumber",
            "Olive Oil",
            "Vinegar",
          ],
          instructions: [
            "Grill chicken until cooked.",
            "Chop vegetables.",
            "Mix all together with dressing.",
          ],
        },
        {
          id: 5,
          name: "Spaghetti Bolognese",
          image: "https://images.unsplash.com/photo-1502741338009-cac2772e18bc?w=400&q=80",
          category: "Dinner",
          cookTime: 40,
          ingredients: [
            "Spaghetti",
            "Ground Beef",
            "Tomato Sauce",
            "Onion",
            "Garlic",
            "Olive Oil",
          ],
          instructions: [
            "Cook pasta.",
            "Sauté meat, onion, and garlic.",
            "Add sauce and combine with pasta.",
          ],
        },
      ],
      users: [
        // Example test user - pw: "1234"
        { username: "testuser", password: "1234", favorites: [1, 3] },
      ],
      favoriteRecipes: [],
    };
  },
  computed: {
    // Filter recipes by selected category and search query
    filteredRecipes() {
      let filtered = this.recipes;
      if (this.selectedCategory !== "All") {
        filtered = filtered.filter(
          (r) => r.category === this.selectedCategory
        );
      }
      if (this.searchQuery) {
        const q = this.searchQuery.trim().toLowerCase();
        filtered = filtered.filter(
          (r) =>
            r.name.toLowerCase().includes(q) ||
            r.category.toLowerCase().includes(q) ||
            r.ingredients.some((i) => i.toLowerCase().includes(q))
        );
      }
      return filtered;
    },
    // Dynamic CSS styles for theming
    rootStyle() {
      return {
        '--recipeease-primary': '#4CAF50',
        '--recipeease-secondary': '#FFC107',
        '--recipeease-accent': '#FF5722',
        background: '#fff',
        minHeight: '100vh',
        color: '#232323',
      };
    },
    sidebarStyle() {
      return {
        background: 'var(--recipeease-primary)',
      };
    },
    mainStyle() {
      return {
        background: '#fff',
      };
    },
  },
  methods: {
    // PUBLIC_INTERFACE
    selectCategory(category) {
      this.selectedCategory = category;
      this.showDetail = false;
    },
    // PUBLIC_INTERFACE
    search() {
      this.showDetail = false;
    },
    // PUBLIC_INTERFACE
    openRecipe(recipe) {
      this.recipeDetail = recipe;
      this.showDetail = true;
    },
    // PUBLIC_INTERFACE
    isFavorite(recipeId) {
      return (
        this.isAuthenticated &&
        this.favoriteRecipes.some((r) => r.id === recipeId)
      );
    },
    // PUBLIC_INTERFACE
    toggleFavorite(recipeId) {
      if (!this.isAuthenticated) return;
      const user = this.currentUser;
      const favorites = user.favorites || [];
      if (favorites.includes(recipeId)) {
        user.favorites = favorites.filter((id) => id !== recipeId);
      } else {
        user.favorites = [...favorites, recipeId];
      }
      this.updateFavorites();
    },
    // PUBLIC_INTERFACE
    updateFavorites() {
      if (!this.isAuthenticated || !this.currentUser) {
        this.favoriteRecipes = [];
        return;
      }
      this.favoriteRecipes = this.recipes.filter((r) =>
        this.currentUser.favorites?.includes(r.id)
      );
    },
    // PUBLIC_INTERFACE
    login() {
      // Data validation: basic auth + trim
      const username = this.authForm.username.trim();
      const password = this.authForm.password.trim();
      if (!username || !password) {
        alert("Username and password required.");
        return;
      }
      const user = this.users.find(
        (u) => u.username === username && u.password === password
      );
      if (user) {
        this.isAuthenticated = true;
        this.currentUser = user;
        this.showAuthModal = false;
        this.authForm.username = "";
        this.authForm.password = "";
        this.isRegisterMode = false;
        this.updateFavorites();
      } else {
        alert("Invalid credentials.");
      }
    },
    // PUBLIC_INTERFACE
    register() {
      const username = this.authForm.username.trim();
      const password = this.authForm.password.trim();
      if (!username || !password) {
        alert("Username and password required.");
        return;
      }
      if (password.length < 4) {
        alert("Password must be at least 4 characters.");
        return;
      }
      if (this.users.find((u) => u.username === username)) {
        alert("Username already exists.");
        return;
      }
      const newUser = { username, password, favorites: [] };
      this.users.push(newUser);
      this.isAuthenticated = true;
      this.currentUser = newUser;
      this.updateFavorites();
      this.showAuthModal = false;
      this.authForm.username = "";
      this.authForm.password = "";
      this.isRegisterMode = false;
    },
    // PUBLIC_INTERFACE
    logout() {
      this.isAuthenticated = false;
      this.currentUser = null;
      this.favoriteRecipes = [];
      this.showDetail = false;
    },
  },
  watch: {
    currentUser: {
      handler() {
        this.updateFavorites();
      },
      immediate: true,
    },
    recipes: {
      handler() {
        this.updateFavorites();
      },
      deep: true,
    },
  },
  mounted() {
    // On startup, update favorites if already authenticated
    this.updateFavorites();
  },
};
</script>

<style scoped>
.recippeease-container {
  display: flex;
  flex-direction: row;
  min-height: 100vh;
  font-family: 'Inter', 'Segoe UI', Arial, sans-serif;
}

.sidebar {
  flex: 0 0 230px;
  display: flex;
  flex-direction: column;
  padding: 1.5rem 1rem 0 1rem;
  min-height: 100vh;
  box-shadow: 0 0 7px 0 rgba(30, 30, 30, 0.04), 0 1px 2px rgba(0,0,0,0.05);
}

.logo {
  font-size: 2rem;
  margin-bottom: 1.4rem;
  display: flex;
  flex-direction: row;
  align-items: center;
  color: #fff;
}
.logo-title {
  margin-left: 0.6rem;
  font-weight: bold;
  font-size: 1.3rem;
  letter-spacing: 0.02em;
}

.category-list {
  flex: 1 1 auto;
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
  margin-bottom: 2rem;
}
.category-btn {
  background: #fff6;
  border: none;
  color: #fff;
  font-size: 1rem;
  padding: 0.45rem 1.2rem;
  margin-right: 0.6rem;
  border-radius: 5px;
  text-align: left;
  cursor: pointer;
  transition: background 0.18s;
  outline: none;
}
.category-btn.active, .category-btn:focus {
  background: var(--recipeease-accent);
  color: #fff;
  font-weight: bold;
}
.category-btn:hover:not(.active) {
  background: var(--recipeease-secondary);
  color: #212121;
}
.favorites {
  margin-bottom: 1.4rem;
  color: #fff;
}
.favorites ul {
  padding-left: 18px;
  margin: 0.2rem 0 0 0;
  font-size: 0.98rem;
}
.sidebar-auth {
  margin-top: 2rem;
  text-align: center;
}
.auth-btn {
  background: #fff;
  color: var(--recipeease-primary);
  padding: 0.55rem 1.6rem;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  font-size: 1rem;
  transition: background 0.16s;
}
.auth-btn:hover {
  background: var(--recipeease-secondary);
}
.user-profile {
  color: #fff;
  display: flex;
  flex-direction: column;
  align-items: center;
  font-size: 1rem;
}
.user-icon {
  font-size: 1.5rem;
  margin-bottom: 0.2rem;
}
.logout-btn {
  background: #fff2;
  border: none;
  color: #fff;
  padding: 0.18rem 0.8rem;
  border-radius: 4px;
  margin-top: 0.2rem;
  font-size: 0.95rem;
  cursor: pointer;
}
.logout-btn:hover {
  background: var(--recipeease-accent);
  color: #fff;
}

.main-content {
  flex: 1 1 auto;
  padding: 2.3rem 2.1rem 2.1rem 2.1rem;
  background: #fff;
  min-width: 0;
  /* For responsive transition: */
  transition: padding 0.25s;
}
@media (max-width: 840px) {
  .recipeease-container {
    flex-direction: column;
  }
  .sidebar {
    flex-direction: row;
    flex-wrap: wrap;
    min-height: 1px;
    width: 100vw;
    padding: 1rem 0.2rem 1rem 0.4rem;
  }
  .main-content {
    padding: 1rem 0.6rem;
  }
}

.recippeease-header {
  display: flex;
  align-items: center;
  border-bottom: 2px solid var(--recipeease-primary);
  margin-bottom: 2.1rem;
  padding-bottom: 1rem;
  flex-wrap: wrap;
  gap: 0.8rem;
}

.search-bar {
  flex: 1 1 220px;
  font-size: 1.08rem;
  padding: 0.5rem 1.1rem;
  margin-left: 0;
  border: 1.5px solid var(--recipeease-primary);
  border-radius: 6px;
  outline: none;
  transition: border 0.18s;
}
.search-bar:focus {
  border-color: var(--recipeease-accent);
}

.recipe-list {
  margin-top: 1.4rem;
}
.recipes-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(210px, 1fr));
  gap: 1.1rem;
}
.recipe-card {
  background: #fafafc;
  border: 1.2px solid #ececec;
  border-radius: 10px;
  padding: 1rem 1rem 1.2rem 1rem;
  cursor: pointer;
  transition: box-shadow 0.18s, border 0.18s;
  box-shadow: 0 1.5px 10px rgba(0,0,0,0.04);
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
}
.recipe-card:hover {
  box-shadow: 0 4px 22px rgba(76,175,80,0.15);
  border-color: var(--recipeease-primary);
}
.recipe-img {
  width: 98%;
  height: 120px;
  object-fit: cover;
  border-radius: 8px;
  margin-bottom: 0.9rem;
  border: 1px solid #e0e0e0;
}
.recipe-card h4 {
  margin: 0 0 0.3rem 0;
  font-size: 1.13rem;
  font-weight: 600;
  color: #333;
}
.meta {
  font-size: 0.97rem;
  color: #555;
  display: flex;
  flex-direction: row;
  gap: 0.8rem;
  margin-bottom: 0.1rem;
}
.favorite-btn {
  position: absolute;
  top: 11px;
  right: 12px;
  background: transparent;
  border: none;
  font-size: 1.29rem;
  cursor: pointer;
  color: var(--recipeease-accent);
  transition: color 0.13s;
}
.favorite-btn.filled {
  color: var(--recipeease-primary);
}
.favorite-btn:focus, .favorite-btn:hover {
  color: var(--recipeease-accent);
}
.no-results {
  grid-column: 1 / -1;
  text-align: center;
  color: #888;
  padding: 2.2rem;
  font-size: 1.13rem;
}

.recipe-detail {
  max-width: 540px;
  margin: 0 auto;
  background: #fbfbff;
  padding: 1.3rem 1.7rem 2.1rem 1.7rem;
  border-radius: 12px;
  box-shadow: 0 2px 22px rgba(76,175,80,0.05);
}
.recipe-detail h2 {
  font-size: 1.8rem;
  margin-bottom: 0.7rem;
}
.detail-meta {
  margin-bottom: 1.4rem;
  display: flex;
  flex-direction: row;
  gap: 1.2rem;
  align-items: center;
}
.detail-meta .category {
  background: var(--recipeease-secondary);
  color: #734900;
  padding: 0.29rem 0.7rem;
  border-radius: 5px;
}
.detail-meta .time {
  background: #f7f7f7;
  border-radius: 6px;
  padding: 0.23rem 0.7rem;
  color: #232323;
}
.detail-meta .favorite-btn {
  position: static;
  margin-left: auto;
}

.recipe-detail ul,
.recipe-detail ol {
  margin-left: 1.2rem;
  margin-bottom: 1rem;
}

.back-btn {
  background: var(--recipeease-primary);
  color: #fff;
  border: none;
  border-radius: 5px;
  padding: 0.4rem 1rem;
  margin-bottom: 1.2rem;
  cursor: pointer;
  font-weight: 500;
}
.back-btn:hover, .modal-action-btn:hover, .switch-btn:hover {
  background: var(--recipeease-accent);
  color: #fff;
}

.modal-backdrop {
  position: fixed;
  inset: 0;
  background: #000c;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10000;
}
.auth-modal {
  background: #fff;
  padding: 2.2rem 2.2rem 1.5rem 2.2rem;
  border-radius: 12px;
  box-shadow: 0 2px 48px rgba(40,40,40,0.18);
  min-width: 280px;
  max-width: 95vw;
  position: relative;
}
.auth-modal h3 {
  margin: 0 0 1.2rem 0;
  font-size: 1.5rem;
  font-weight: bold;
  color: var(--recipeease-primary);
}
.auth-modal form {
  display: flex;
  flex-direction: column;
  gap: 1.0rem;
  margin-bottom: 0.7rem;
}
.auth-modal input[type="text"],
.auth-modal input[type="password"] {
  border: 1.2px solid #ccc;
  border-radius: 6px;
  padding: 0.53rem 1rem;
  font-size: 1.07rem;
}
.auth-modal input:focus {
  border-color: var(--recipeease-primary);
  outline: none;
}
.modal-action-btn {
  background: var(--recipeease-primary);
  color: #fff;
  border: none;
  border-radius: 5px;
  padding: 0.5rem 0.7rem;
  font-size: 1.06rem;
  font-weight: 600;
  cursor: pointer;
  margin-top: 0.5rem;
}
.switch-auth-mode {
  text-align: center;
  margin-top: 0.3rem;
}
.switch-btn {
  background: none;
  border: none;
  color: var(--recipeease-accent);
  font-size: 1rem;
  cursor: pointer;
  margin: 0;
  padding: 0;
  text-decoration: underline;
}

.modal-close {
  position: absolute;
  top: 12px;
  right: 14px;
  background: none;
  border: none;
  font-size: 1.33rem;
  cursor: pointer;
  color: #999;
}
.modal-close:hover { color: var(--recipeease-accent); }
</style>
