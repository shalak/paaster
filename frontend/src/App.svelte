<script lang="ts">
  import "./assets/style.css";
  import "./assets/icons/1.3.0/css/line-awesome.min.css";
  import { Router, link, Route } from "svelte-navigator";
  import { Modals, closeModal } from "svelte-modals";
  import { isLoading, _ } from "svelte-i18n";
  import { tooltip } from "@svelte-plugins/tooltips";

  import { Toaster } from "svelte-french-toast";
  import { Loading } from "@tadashi/svelte-loading";
  import CreatePaste from "./routes/CreatePaste.svelte";
  import LazyRoute from "./components/LazyRoute.svelte";
  import PageLoading from "./components/PageLoading.svelte";

  import "./i18n";
</script>

{#if !$isLoading}
  <Router primary={false}>
    <nav>
      <a href="/" use:link
        ><h1>
          {import.meta.env.VITE_NAME ? import.meta.env.VITE_NAME : "paaster"}
        </h1></a
      >
      <ul>
        <li>
          <a
            use:tooltip={{ content: "Star us on Github", position: "bottom" }}
            href="https://github.com/WardPearce/paaster"
            target="_blank"
            rel="noopener noreferrer"
          >
            <i class="lab la-github" /></a
          >
        </li>
        <li>
          <a href="/pastes" use:link class="button"
            ><i class="lab la-buffer" />{$_("saved_pastes")}</a
          >
        </li>
      </ul>
    </nav>

    <Route path="/">
      <CreatePaste />
    </Route>
    <LazyRoute
      path="/pastes"
      delayMs={500}
      component={() => import("./routes/StorePastes.svelte")}
    >
      <PageLoading />
    </LazyRoute>
    <LazyRoute
      path="/privacy-policy"
      delayMs={500}
      component={() => import("./routes/PrivacyPolicy.svelte")}
    >
      <PageLoading />
    </LazyRoute>
    <LazyRoute
      path="/terms-of-service"
      delayMs={500}
      component={() => import("./routes/TermsOfService.svelte")}
    >
      <PageLoading />
    </LazyRoute>
    <LazyRoute
      path="/:pasteId"
      delayMs={500}
      component={() => import("./routes/ViewPaste.svelte")}
    >
      <PageLoading />
    </LazyRoute>
  </Router>
{/if}
<Loading
  animation="Jelly"
  color="var(--lighterBg)"
  --tadashi_svelte_loading_background_color="var(--darkBgTrans)"
  --tadashi_svelte_loading_zindex="1011"
/>

<Toaster toastOptions={{ className: "toast" }} />

<Modals>
  <div
    slot="backdrop"
    class="backdrop"
    on:click={closeModal}
    on:keydown={() => {}}
  />
</Modals>
