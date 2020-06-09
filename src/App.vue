<template>
  <div id="app">
    <div id="nav">
      <router-link to="/">Home</router-link> |
      <router-link to="/about">About</router-link> |
      <router-link to="/bar-chart">Bar Chart</router-link>
    </div>
    <router-view />
  </div>
</template>

<style>
#app {
  color: #2c3e50;
  font-family: "Inter var", sans-serif;
  display: grid;
  grid-template-areas:
    "nav"
    "content";
  gap: 20px;

  /* height: 100vh; */
}

#nav {
  grid-area: nav;
  justify-self: center;
  padding: 30px;
  background-color: lightblue;
}

#nav a {
  font-weight: bold;
  color: #2c3e50;
  margin: 0 7px;
}

#nav a.router-link-exact-active {
  color: #42b983;
}
html {
  box-sizing: border-box;
}
*,
*:before,
*:after {
  box-sizing: inherit;
}
@font-face {
  font-family: "Inter var";
  font-display: swap;
  font-weight: 100 900;
  font-style: normal;
  font-named-instance: "Regular";
  src: url("./static/fonts/Inter.var.woff2") format("woff2");
}
@media all and (max-width: 700px) {
}
</style>
