<script>
  import _ from 'lodash-es'
  import { tick } from 'svelte'
  import { fade } from 'svelte/transition'
  import { afterNavigate } from '$app/navigation'
  import { find, isEqual, cloneDeep } from 'lodash-es'
  import Block from './Layout/Block.svelte'
  import Spinner from '../../ui/misc/Spinner.svelte'
  import { code as siteCode } from '../../stores/data/site'
  import { locale } from '../../stores/app/misc'
  import { updatePreview } from '../../stores/actions'
  import {
    id as pageID,
    url as pageURL,
    fields as pageFields,
    code as pageCode,
    content as pageContent,
  } from '../../stores/app/activePage'
  import sections from '../../stores/data/sections'
  import { processCode, processCSS, wrapInStyleTags } from '../../utils'
  import { getPageData } from '../../stores/helpers'
  import en from '../../languages/en.json'
  import { init, addMessages } from 'svelte-i18n'
  import { supabase } from '$lib/supabase'
  import { track, locked_blocks } from '$lib/realtime'
  import { invalidate } from '$app/navigation'
  import { browser } from '$app/environment'

  export let data

  let html_head = ''
  let html_below = ''

  if (browser) {
    supabase
      .channel('schema-db-changes')
      .on(
        'postgres_changes',
        {
          event: 'UPDATE',
          schema: 'public',
          table: 'sections',
          filter: `page=eq.${data.page.id}`,
        },
        () => {
          invalidate('app:data')
        }
      )
      .subscribe()
  }

  addMessages('en', en)
  init({
    fallbackLocale: 'en',
    initialLocale: 'en',
  })

  let element

  $: set_page_content(data.page)
  async function set_page_content(page_data) {
    if (!page_data) return
    await tick()
    $sections = data.sections

    $pageID = page_data.id
    $pageURL = page_data.url
    $pageFields = page_data.fields
    $pageCode = page_data.code
    $pageContent = page_data.content

    if (page_mounted) updatePreview()
  }

  const cached = { pageCode: null, siteCode: null }
  let latest_run
  $: set_page_html($pageCode, $siteCode)
  async function set_page_html(pageCode, siteCode) {
    if (
      isEqual(pageCode, cached.pageCode) &&
      isEqual(siteCode, cached.siteCode)
    )
      return

    const this_run = Date.now()

    cached.pageCode = cloneDeep(pageCode)
    cached.siteCode = cloneDeep(siteCode)
    const css = await processCSS(siteCode.css + pageCode.css)

    // workaround to prevent older css from overwriting newer css
    if (latest_run > this_run) return
    latest_run = this_run

    const data = getPageData({})
    const [head, below] = await Promise.all([
      processCode({
        component: {
          html: `<svelte:head>
            ${siteCode.html.head}${pageCode.html.head}
            ${wrapInStyleTags(css)}
          </svelte:head>`,
          css: '',
          js: '',
          data,
        },
      }),
      processCode({
        component: {
          html: siteCode.html.below + pageCode.html.below,
          css: '',
          js: '',
          data,
        },
      }),
    ])
    html_head = !head.error ? head.head : ''
    html_below = !below.error ? below.html : ''
  }

  // Fade in page when all components mounted
  let page_mounted = false
  $: page_is_empty = $sections.length === 0

  // detect when all sections are mounted
  let sections_mounted = 0
  $: if (sections_mounted === $sections.length && sections_mounted !== 0) {
    page_mounted = true
  }

  afterNavigate(() => {
    page_mounted = false
    sections_mounted = 0
  })

  async function lock_block(block_id) {
    track({
      active_block: block_id,
      user: data.user,
    })
  }

  function unlock_block() {
    // workaround to prevent issue when unlocking immediately before locking when switching from one block to another
    setTimeout(() => {
      track({
        active_block: null,
      })
    }, 100)
  }
</script>

<svelte:head>
  {@html html_head}
</svelte:head>

{#if !page_mounted && $sections.length > 0}
  <div class="spinner-container" out:fade={{ duration: 200 }}>
    <Spinner />
  </div>
{/if}

<div id="page" bind:this={element} class:fadein={page_mounted} lang={$locale}>
  {#each $sections.sort((a, b) => a.index - b.index) as block, i (block.id)}
    <Block
      {i}
      locked={find($locked_blocks, ['block_id', block.id])}
      {block}
      on:lock={() => lock_block(block.id)}
      on:unlock={() => unlock_block()}
      on:mount={() => {
        sections_mounted++
      }}
    />
  {/each}
</div>
{@html html_below || ''}

{#if page_is_empty}
  <div class="empty-state">This is an empty page</div>
{/if}

<style lang="postcss">
  .spinner-container {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 5;

    --Spinner-color: var(--primo-color-brand);
    --Spinner-color-opaque: rgba(248, 68, 73, 0.2);
  }
  #page {
    transition: 0.2s opacity;
    opacity: 0;
    border-top: 0;
    height: calc(100vh - 54px);
    overflow: auto;
  }
  #page.fadein {
    opacity: 1;
  }
  .empty-state {
    position: absolute;
    inset: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    color: var(--color-gray-4);
    pointer-events: none;
    z-index: -2;
    font-family: Inter, sans-serif;
    color: #999;
    z-index: 1;
    text-align: center;
  }
</style>
