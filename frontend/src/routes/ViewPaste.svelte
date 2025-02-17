<script lang="ts">
  import { navigate } from "svelte-navigator";
  import { HighlightAuto, LineNumbers } from "svelte-highlight";
  import rosPine from "svelte-highlight/styles/ros-pine";
  import { get } from "svelte/store";
  import { onMount } from "svelte";
  import sodium from "libsodium-wrappers";
  import { acts } from "@tadashi/svelte-loading";
  import toast from "svelte-french-toast";
  import { openModal } from "svelte-modals";
  import Mousetrap from "mousetrap";
  import Select from "svelte-select";
  import { _ } from "svelte-i18n";

  import { pasteStore } from "../stores";
  import { paasterClient } from "../lib/client";
  import type { PasteModel } from "../lib/client/models/PasteModel";
  import { ApiError } from "../lib/client/core/ApiError";
  import { deletePaste, getPaste, savePaste } from "../lib/savedPaste";

  export let pasteId: string;

  let ownerSecret = "";
  const [b64EncodedRawKey, givenOwnerSecret]: string[] = location.hash
    .substring(1)
    .split("&ownerSecret=");

  // Remove ownerSecret out of URL ASAP if provided.
  if (typeof givenOwnerSecret !== "undefined") {
    location.hash = `#${b64EncodedRawKey}`;
  }

  let isSaved = false;
  let rawCode = "";
  let pasteCreated: number;
  let timePeriods = [
    { value: null, label: $_("paste_actions.expire.periods.never") },
    { value: -1, label: $_("paste_actions.expire.periods.being_viewed") },
    {
      value: 0.08333,
      label: `5 ${$_("paste_actions.expire.periods.minutes")}`,
    },
    { value: 0.25, label: `15 ${$_("paste_actions.expire.periods.minutes")}` },
    { value: 0.5, label: `30 ${$_("paste_actions.expire.periods.minutes")}` },
    { value: 1, label: `1 ${$_("paste_actions.expire.periods.hour")}` },
    { value: 2, label: `2 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 3, label: `3 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 4, label: `4 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 5, label: `5 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 6, label: `6 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 7, label: `7 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 8, label: `8 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 9, label: `9 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 10, label: `10 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 11, label: `11 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 12, label: `12 ${$_("paste_actions.expire.periods.hours")}` },
    { value: 24, label: `1 ${$_("paste_actions.expire.periods.day")}` },
    { value: 48, label: `2 ${$_("paste_actions.expire.periods.days")}` },
    { value: 72, label: `3 ${$_("paste_actions.expire.periods.days")}` },
    { value: 96, label: `4 ${$_("paste_actions.expire.periods.days")}` },
    { value: 120, label: `5 ${$_("paste_actions.expire.periods.days")}` },
    { value: 144, label: `6 ${$_("paste_actions.expire.periods.days")}` },
    { value: 168, label: `1 ${$_("paste_actions.expire.periods.week")}` },
    { value: 336, label: `2 ${$_("paste_actions.expire.periods.weeks")}` },
    { value: 504, label: `3 ${$_("paste_actions.expire.periods.weeks")}` },
    { value: 730, label: `1 ${$_("paste_actions.expire.periods.month")}` },
    { value: 1461, label: `2 ${$_("paste_actions.expire.periods.months")}` },
    { value: 2192, label: `3 ${$_("paste_actions.expire.periods.months")}` },
  ];
  let selectedTime = null;

  acts.show(true);

  function renamePaste() {
    openModal(() => import("../components/RenamePaste.svelte"), {
      pasteId: pasteId,
    });
  }

  function setAccessCode() {
    openModal(() => import("../components/SetAccessCode.svelte"), {
      pasteId: pasteId,
      ownerSecret: ownerSecret,
      b64EncodedRawKey: b64EncodedRawKey,
    });
  }

  function generateQRCode() {
    openModal(() => import("../components/QRCode.svelte"), {
      pasteId: pasteId,
      b64EncodedRawKey: b64EncodedRawKey,
    });
  }

  async function shareLinkToClipboard() {
    await navigator.clipboard.writeText(window.location.href);
    toast.success($_("paste_actions.share.success"));
  }

  async function expireAfter(event: {
    detail: { value: number; label: string };
  }) {
    await toast.promise(
      paasterClient.default.controllerPasteUpdatePaste(pasteId, ownerSecret, {
        expires_in_hours: event.detail.value,
      }),
      {
        loading: $_("paste_actions.expire.loading"),
        success: $_("paste_actions.expire.success", {
          values: { period: event.detail.label },
        }),
        error: $_("paste_actions.expire.error"),
      }
    );
  }

  async function deletePasteCall() {
    try {
      await toast.promise(
        paasterClient.default.controllerPasteDeletePaste(pasteId, ownerSecret),
        {
          loading: $_("paste_actions.delete.loading"),
          error: $_("paste_actions.delete.error"),
          success: $_("paste_actions.delete.success"),
        }
      );
      await deletePaste(pasteId);
      navigate("/", { replace: true });
    } catch {
      toast.error("Unable to delete paste");
    }
  }

  async function download() {
    const anchor = document.createElement("a");
    const url = window.URL.createObjectURL(
      new Blob([rawCode], { type: "octet/stream" })
    );
    anchor.href = url;
    anchor.click();
    window.URL.revokeObjectURL(url);
  }

  async function savePasteLocal() {
    await savePaste(pasteId, b64EncodedRawKey, pasteCreated);
    toast.success($_("paste_actions.save.success"));
    isSaved = true;
  }

  async function copyToClipboard() {
    await navigator.clipboard.writeText(rawCode);
    toast.success($_("paste_actions.clipboard.success"));
  }

  async function loadPaste(accessCode?: string) {
    acts.show(true);

    try {
      const savedPaste = await getPaste(pasteId);
      if (savedPaste.ownerSecret) ownerSecret = savedPaste.ownerSecret;
      isSaved = true;
    } catch {}

    // If user just created paste,
    // avoid needing to download & decrypt paste for speed reasons.
    let storedPaste = get(pasteStore);
    if (storedPaste !== "") {
      rawCode = storedPaste;
      pasteStore.set("");
      acts.show(false);
      return;
    }

    await sodium.ready;

    if (b64EncodedRawKey === "") {
      toast.error($_("view.no_key"));
      acts.show(false);
      navigate("/", { replace: true });
      return;
    }

    let rawSecretKey: Uint8Array;
    try {
      rawSecretKey = sodium.from_base64(
        b64EncodedRawKey,
        sodium.base64_variants.URLSAFE_NO_PADDING
      );
    } catch {
      toast.error($_("view.invalid_format"));
      acts.show(false);
      navigate("/", { replace: true });
      return;
    }

    let paste: PasteModel;
    try {
      paste = await paasterClient.default.controllerPasteGetPaste(
        pasteId,
        accessCode
      );
    } catch (error) {
      if (error instanceof ApiError) {
        // Delete paste from local storage if no longer exists on server.
        if (error.status === 404) {
          await deletePaste(pasteId);
          toast.error($_("view.404"));
        } else if (error.status == 401) {
          toast.error(error.body.detail);
          openModal(() => import("../components/ProvideAccessCode.svelte"), {
            loadPasteFunc: loadPaste,
            b64EncodedRawKey: b64EncodedRawKey,
          });
          acts.show(false);
          return;
        } else {
          toast.error(error.body.detail);
        }
      } else if (error instanceof Error) toast.error(error.toString());
      acts.show(false);
      navigate("/", { replace: true });
      return;
    }

    if (typeof givenOwnerSecret !== "undefined") {
      ownerSecret = givenOwnerSecret;
      try {
        await savePaste(pasteId, b64EncodedRawKey, paste.created, ownerSecret);
        isSaved = true;
      } catch {}
    }

    if (ownerSecret !== "" && paste.expires_in_hours !== null) {
      // Allows us to change the period label in the future.
      timePeriods.forEach((time) => {
        if (time.value === paste.expires_in_hours) {
          selectedTime = time;
          return true;
        }
      });
    }

    let response: Response;
    try {
      response = await fetch(paste.download_url);
    } catch {
      toast.error($_("view.cdn_down"));
      acts.show(false);
      navigate("/", { replace: true });
      return;
    }

    try {
      rawCode = new TextDecoder("utf8").decode(
        sodium.crypto_aead_xchacha20poly1305_ietf_decrypt(
          null,
          new Uint8Array(await response.arrayBuffer()),
          null,
          sodium.from_base64(
            paste.iv,
            sodium.base64_variants.URLSAFE_NO_PADDING
          ),
          rawSecretKey
        )
      );
    } catch (error) {
      if (error instanceof Error) toast.error(error.toString());
      acts.show(false);
      navigate("/", { replace: true });
      return;
    }

    pasteCreated = Number(paste.created);

    acts.show(false);

    Mousetrap.bind(["command+a", "ctrl+a"], () => {
      copyToClipboard();
      return false;
    });
    Mousetrap.bind(["command+x", "ctrl+x"], () => {
      shareLinkToClipboard();
      return false;
    });
    Mousetrap.bind(["command+s", "ctrl+s"], () => {
      download();
      return false;
    });
  }

  onMount(async () => {
    await loadPaste();
  });
</script>

<svelte:head>
  {@html rosPine}
</svelte:head>

{#if ownerSecret !== ""}
  <main>
    <section>
      <h3>owner panel</h3>
      <div class="owner-panel">
        <button on:click={renamePaste}
          ><i class="las la-pencil-alt" />{$_(
            "paste_actions.rename.button"
          )}</button
        >
        <button on:click={shareLinkToClipboard}
          ><i class="las la-share" />{$_("paste_actions.share.button")}</button
        >
        <button on:click={generateQRCode}
          ><i class="las la-qrcode" />{$_(
            "paste_actions.qr_code.button"
          )}</button
        >
        <button on:click={setAccessCode}
          ><i class="las la-key" />{$_(
            "paste_actions.access_code.button"
          )}</button
        >
        <Select
          items={timePeriods}
          clearable={false}
          placeholder={$_("paste_actions.expire.button")}
          on:change={expireAfter}
          bind:value={selectedTime}
        />
        <button class="danger" on:click={deletePasteCall}
          ><i class="las la-trash" />{$_("paste_actions.delete.button")}</button
        >
      </div>
    </section>
  </main>
{/if}

<footer>
  <button on:click={download}
    ><i class="las la-download" />{$_("paste_actions.download.button")}</button
  >
  <button on:click={copyToClipboard}
    ><i class="las la-copy" />{$_("paste_actions.clipboard.button")}</button
  >

  {#if !isSaved}
    <button on:click={savePasteLocal}
      ><i class="las la-save" />{$_("paste_actions.save.button")}</button
    >
  {/if}
</footer>

{#if rawCode !== ""}
  <div class="content">
    <HighlightAuto code={rawCode} let:highlighted>
      <LineNumbers {highlighted} />
    </HighlightAuto>
  </div>
{/if}

<style>
  .content {
    margin-top: 1em;
    margin-bottom: 20em;
  }

  .owner-panel {
    margin-top: 0.5em;
    display: flex;
    column-gap: 1em;
  }

  @media screen and (max-width: 1200px) {
    .owner-panel {
      flex-direction: column;
      row-gap: 1em;
    }
  }
</style>
