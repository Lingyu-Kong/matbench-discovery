<script lang="ts">
  import type { ModelData } from '$lib'
  import {
    MetricsTable,
    model_is_compliant,
    MODEL_METADATA,
    TableColumnToggleMenu,
  } from '$lib'
  import { METADATA_COLS, METRICS_COLS } from '$lib/metrics'
  import Readme from '$root/readme.md'
  import KappaNote from '$site/src/routes/kappa-note.md'
  import LandingPageFigs from '$site/src/routes/landing-page-figs.md'
  import Icon from '@iconify/svelte'
  import { pretty_num } from 'elementari'
  import { Toggle, Tooltip } from 'svelte-zoo'

  let n_wbm_stable_uniq_protos = 32_942
  let n_wbm_uniq_protos = 215_488

  let show_non_compliant: boolean = false
  let show_energy_only: boolean = false

  // Default column visibility
  let visible_cols: Record<string, boolean> = {
    ...Object.fromEntries(
      [...METRICS_COLS, ...METADATA_COLS].map((col) => [col.label, true]),
    ),
    TPR: false,
    TNR: false,
    RMSE: false,
  }

  $: best_model = MODEL_METADATA.reduce((best, md: ModelData) => {
    const best_F1 = best.metrics?.discovery?.full_test_set?.F1 ?? 0
    const md_F1 = md.metrics?.discovery?.full_test_set?.F1 ?? 0
    if ((!best_F1 || md_F1 > best_F1) && (show_non_compliant || model_is_compliant(md))) {
      return md
    }
    return best
  }, {} as ModelData)

  // Get array of hidden columns
  $: hide_cols = Object.entries(visible_cols)
    .filter(([_, visible]) => !visible)
    .map(([col]) => col)

  let column_panel_open: boolean = false

  const discovery_set_labels = {
    full_test_set: {
      title: `Full Test Set`,
      tooltip: `Metrics computed on the full test set including duplicate structure prototypes`,
    },
    unique_prototypes: {
      title: `Unique Prototypes`,
      tooltip: `Metrics computed only on ~215k unique structure prototypes in WBM determined by matching Aflow-style prototype strings.`,
      link: `https://github.com/janosh/matbench-discovery/blob/fd1dda6c/data/wbm/compile_wbm_test_set.py#L632-L705`,
    },
    most_stable_10k: {
      title: `10k Most Stable`,
      tooltip: `Metrics computed on the 10k structures predicted to be most stable (different for each model)`,
    },
  }
  let discovery_set: keyof typeof discovery_set_labels = `unique_prototypes`
</script>

<Readme>
  <figure style="margin-top: 4em;" slot="metrics-table">
    <div class="discovery-set-toggle">
      {#each Object.entries(discovery_set_labels) as [key, { title, tooltip, link }]}
        <Tooltip text={tooltip} tip_style="z-index: 2; font-size: 0.8em;">
          <button
            class:active={discovery_set === key}
            on:click={() => (discovery_set = key)}
          >
            {title}
            {#if link}
              <a href={link} target="_blank">
                <Icon icon="octicon:info" inline />
              </a>
            {/if}
          </button>
        </Tooltip>
      {/each}
    </div>

    <MetricsTable
      {show_non_compliant}
      {hide_cols}
      {show_energy_only}
      {discovery_set}
      style="width: 100%;"
    />

    <div class="downloads">
      Download table as
      {#each [`PDF`, `SVG`] as file_ext}
        {@const suffix = show_non_compliant ? `` : `-only-compliant`}
        <a
          href="/figs/metrics-table-uniq-protos{suffix}.{file_ext.toLowerCase()}"
          download
        >
          {file_ext}
        </a>
      {/each}
    </div>
    <div class="table-controls">
      <Toggle bind:checked={show_non_compliant} style="gap: 3pt;">
        Show non-compliant models <Tooltip max_width="20em">
          <span slot="tip">
            Models can be non-compliant for multiple reasons<br />
            - closed source (model implementation and/or train/test code)<br />
            - closed weights<br />
            - trained on more than the permissible training set (<a
              href="https://docs.materialsproject.org/changes/database-versions#v2022.10.28"
              >MP v2022.10.28 release</a
            >)<br />
            We still show these models behind a toggle as we expect them<br /> to nonetheless
            provide helpful signals for developing future models.
          </span>
          <Icon icon="octicon:info-16" inline style="padding: 0 3pt;" />
        </Tooltip>&ensp;</Toggle
      >
      <Toggle bind:checked={show_energy_only} style="gap: 3pt;">
        Show energy-only models <Tooltip max_width="12em">
          <span slot="tip">
            Models that only predict energy (E) perform worse<br /> and can't be evaluated
            on force-modeling tasks such as κ<sub>SRME</sub>
          </span>
          <Icon icon="octicon:info-16" inline style="padding: 0 3pt;" />
        </Tooltip>&ensp;</Toggle
      >

      <TableColumnToggleMenu bind:visible_cols bind:column_panel_open />
    </div>

    <figcaption>
      Training size is the number of materials used to train the model. For models trained
      on DFT relaxations, we show the number of distinct frames in parentheses. In cases
      where only the number of frames is known, we report the number of frames as the
      training set size. <code>(N=x)</code> in the Model Params column shows the number of
      estimators if an ensemble was used. DAF = Discovery Acceleration Factor measures how
      many more stable materials a model finds compared to random selection from the test
      set. The unique structure prototypes in the WBM test set have a
      <code>{pretty_num(n_wbm_stable_uniq_protos / n_wbm_uniq_protos, `.1%`)}</code> rate
      of stable crystals, meaning the max possible DAF is
      <code
        >({pretty_num(n_wbm_stable_uniq_protos)} / {pretty_num(n_wbm_uniq_protos)})^−1 ≈
        {pretty_num(n_wbm_uniq_protos / n_wbm_stable_uniq_protos)}</code
      >.
    </figcaption>
  </figure>

  <span slot="model-count">
    {MODEL_METADATA.filter((md) => show_non_compliant || model_is_compliant(md)).length}
  </span>

  <div slot="best-report">
    {#if best_model}
      {@const { model_name, model_key, repo, paper, metrics = {} } = best_model}
      {@const { F1, R2, DAF } = metrics?.discovery?.[discovery_set] ?? {}}

      <a href="/models/{model_key}">{model_name}</a> (<a href={paper}>paper</a>,
      <a href={repo}>code</a>) achieves the highest F1 score of {F1}, R<sup>2</sup> of {R2}
      and a discovery acceleration factor (DAF) of {DAF}
      (i.e. a ~{Number(DAF).toFixed(1)}x higher rate of stable structures compared to
      dummy discovery in the already enriched test set containing 16% stable materials).
    {/if}
  </div>
</Readme>
<KappaNote />

<LandingPageFigs />

<style>
  figure {
    margin: 0;
    display: grid;
    gap: 1ex;
  }
  figcaption {
    font-size: 0.9em;
    padding: 2pt 6pt;
    background-color: rgba(255, 255, 255, 0.06);
  }
  .discovery-set-toggle {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 5pt;
    margin-bottom: 5pt;
  }
  .discovery-set-toggle button {
    padding: 4px 8px;
    border: 1px solid rgba(255, 255, 255, 0.05);
    background: transparent;
  }
  .discovery-set-toggle button:hover {
    background: rgba(255, 255, 255, 0.05);
  }
  .discovery-set-toggle button.active {
    background: rgba(255, 255, 255, 0.1);
  }
  div.downloads {
    display: flex;
    gap: 1ex;
    justify-content: center;
    margin-block: 1ex;
  }
  div.downloads a {
    background-color: rgba(255, 255, 255, 0.1);
    padding: 0 6pt;
    border-radius: 4pt;
  }
  div.table-controls {
    display: flex;
    flex-wrap: wrap;
    gap: 5pt;
    place-content: center;
    margin: 3pt auto;
  }
  :is(figure[slot='metrics-table'], .column-menu) :global(:is(sub, sup)) {
    transform: translate(-3pt, 6pt);
    font-size: 0.7em;
  }
</style>
