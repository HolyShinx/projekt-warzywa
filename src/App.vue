<script setup>
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <v-app>
    <v-app-bar 
      app 
      color="green darken-2" 
      dark 
      elevation="4"
      height="70"
      class="navbar-enhanced"
    >
      <!-- Logo/Brand Section -->
      <div class="brand-section">
        <v-icon class="brand-icon mr-3" size="32">mdi-leaf</v-icon>
        <v-toolbar-title class="brand-title">
          Projekt Warzywa
        </v-toolbar-title>
      </div>

      <v-spacer></v-spacer>

      <!-- Navigation Links -->
      <div class="nav-links">
        <RouterLink 
          to="/" 
          class="nav-link"
          active-class="nav-link-active"
        >
          <v-icon class="nav-icon" size="20">mdi-home</v-icon>
          <span class="nav-text">Home</span>
        </RouterLink>
        
        <RouterLink 
          to="/Tables" 
          class="nav-link"
          active-class="nav-link-active"
        >
          <v-icon class="nav-icon" size="20">mdi-table</v-icon>
          <span class="nav-text">Warzywa</span>
        </RouterLink>
      </div>

      <!-- Optional: Mobile menu button for responsive design -->
      <v-app-bar-nav-icon 
        class="d-md-none"
        @click="drawer = !drawer"
      ></v-app-bar-nav-icon>
    </v-app-bar>

    <!-- Optional: Navigation drawer for mobile -->
    <v-navigation-drawer
      v-model="drawer"
      temporary
      class="d-md-none"
    >
      <v-list>
        <v-list-item
          v-for="item in navItems"
          :key="item.title"
          :to="item.route"
          router
        >
          <v-list-item-icon>
            <v-icon>{{ item.icon }}</v-icon>
          </v-list-item-icon>
          <v-list-item-content>
            <v-list-item-title>{{ item.title }}</v-list-item-title>
          </v-list-item-content>
        </v-list-item>
      </v-list>
    </v-navigation-drawer>

    <v-main>
      <v-container fluid class="main-content">
        <RouterView />
      </v-container>
    </v-main>
  </v-app>
</template>

<script>
export default {
  data() {
    return {
      drawer: false,
      navItems: [
        { title: 'Home', route: '/', icon: 'mdi-home' },
        { title: 'Warzywa', route: '/Tables', icon: 'mdi-table' }
      ]
    }
  }
}
</script>

<style scoped>
/* Enhanced Navigation Bar Styling */
.navbar-enhanced {
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%) !important;
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

/* Brand Section */
.brand-section {
  display: flex;
  align-items: center;
  padding: 0 16px;
}

.brand-icon {
  color: #81c784 !important;
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.2));
}

.brand-title {
  font-size: 1.5rem !important;
  font-weight: 600 !important;
  background: linear-gradient(45deg, #ffffff, #e8f5e8);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* Navigation Links Container */
.nav-links {
  display: flex;
  gap: 8px;
  padding: 0 16px;
}

/* Individual Navigation Links */
.nav-link {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 20px;
  margin: 0 4px;
  text-decoration: none;
  color: rgba(255, 255, 255, 0.9) !important;
  border-radius: 25px;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
  font-weight: 500;
}

/* Hover Effects */
.nav-link:hover {
  background: rgba(255, 255, 255, 0.2);
  color: #ffffff !important;
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.15);
  border-color: rgba(255, 255, 255, 0.3);
}

.nav-link:hover .nav-icon {
  transform: scale(1.1);
  color: #81c784 !important;
}

/* Active State */
.nav-link-active {
  background: rgba(255, 255, 255, 0.25) !important;
  color: #ffffff !important;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
  border-color: rgba(255, 255, 255, 0.4);
}

.nav-link-active .nav-icon {
  color: #81c784 !important;
}

/* Navigation Icons */
.nav-icon {
  transition: all 0.3s ease;
  color: rgba(255, 255, 255, 0.8) !important;
}

/* Navigation Text */
.nav-text {
  font-size: 0.95rem;
  letter-spacing: 0.5px;
  transition: all 0.3s ease;
}

/* Responsive Design */
@media (max-width: 960px) {
  .nav-links {
    display: none;
  }
  
  .brand-title {
    font-size: 1.2rem !important;
  }
}

@media (min-width: 961px) {
  .v-app-bar-nav-icon {
    display: none !important;
  }
}

/* Main Content Styling */
.main-content {
  padding: 2rem 1rem;
  min-height: calc(100vh - 70px);
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
}

@media (min-width: 1024px) {
  .main-content {
    padding: 3rem;
  }
}

/* Animation for smooth transitions */
.nav-link::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.1), transparent);
  transition: left 0.5s;
}

.nav-link:hover::before {
  left: 100%;
}

/* Additional Visual Enhancements */
.v-divider {
  margin: 2rem 0 !important;
}

/* Custom scrollbar for navigation drawer */
.v-navigation-drawer ::-webkit-scrollbar {
  width: 6px;
}

.v-navigation-drawer ::-webkit-scrollbar-track {
  background: #f1f1f1;
}

.v-navigation-drawer ::-webkit-scrollbar-thumb {
  background: #2e7d32;
  border-radius: 3px;
}

.v-navigation-drawer ::-webkit-scrollbar-thumb:hover {
  background: #1b5e20;
}
</style>