<script>
    import "@evidence-dev/tailwind/fonts.css";
    import "../app.css";
    import { EvidenceDefaultLayout } from "@evidence-dev/core-components";
    export let data;
</script>

<EvidenceDefaultLayout
    {data}
    title="Vish"
    hideSidebar={false}
    fullWidth={true}
    neverShowQueries={true}
    builtWithEvidence={true}
>
    <slot slot="content" />
</EvidenceDefaultLayout>
