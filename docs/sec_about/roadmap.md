# Roadmap & Changelog

```mermaid
gantt
    title Roadmap for the remainder of 2025
    dateFormat MM-DD
    axisFormat %d %b
    section Phase 1
    Revive proj          :a1, 08-08, 9d
    Alt-injector         :08-15, 15d
    mermaid              :08-10, 33d
    section Phase 2
    PlantUML             :09-08, 15d
    Graphviz             :09-12, 25d
    Site content         :08-10, 80d
    WCAG*                :08-20, 62d
```

*within the 'WCAG' timeframe: alt text and ARIA landmarks for all three diagram types supported: PlantUML, Graphviz, and Mermaid.

## Changelog

### 2025.12.31 — GitHub Copilot's summary assessment for 2025 August - 2025 year end

* Restarted and restructured the site, got it live on GitHub Pages, and established a clear roadmap for accessibility (WCAG) for diagrams.  
* Moved from experimental JS-only accessibility injection to a more robust mermaid2-based approach and proceeded to add PlantUML and Graphviz support with accessibility metadata.  
* Refactored site JS (theme toggler, metadata injector), improved diagram UX (zoom/lightbox/unified metadata injection), and prepared a static site for deployment.  
* Expanded content considerably: eHealth deep dives (SMART/FHIR), API documentation topics, GenAI landscape, personal/technical memoir, and early experiments on translation automation (DeepL + Vertex AI).  
* Overall: Combined infrastructure (deployment, tooling, accessibility mechanics) with substantive content growth and experimentation (AI-assisted translation/validation).  

**Happy** 2️⃣0️⃣2️⃣6️⃣❣️🙏

### History based on select commits

- 2026.02.25 Updated SMART-ON-FHIR Deep Dive: 'wrap-up', nav polish, removed 'noise', examples more 'copy-paste-safe', bug fixes.
- 2025.12.09 New section with a couple of article translations to explore automation with DeepL API and Vertex AI for 'naturalness check':[Translations](../sec_freeform/sec_transl/index.md). Infrastructure/scripts/workflow: TBD for 2026 😉
- 2025.11.02 New set of topics for exploring[API documentation](../sec_api/index.md) features.
- 2025.11.01 Major refactoring of JS files (theme switcher, metadata injector) and SpotlightJS
- 2025.10.28 Accessibility metadata injection now complete for all three supported diagram types (Mermaid, PlantUML, Graphviz)
- 2025.10.16 a. Graphviz support (no accessibility metadata yet); b. new: a[deep dive into SMART-on-FHIR](../sec_ehealth/sec_deepdive/index.md).
- 2025.10.08 New set of[eHealth-related topics](../sec_ehealth/index.md).
- 2025.10.07 A mid-to-late 2010s[SAML/SSO admin guide](../sec_net-sec/saml-admin.md) for sentimental, aesthetic, and experimentation purposes.
- 2025.10.05 A[topic](../sec_freeform/identity.md) on digital identity and its ~30-year evolution from password chaos to token sprawl.
- 2025.09.19 New topic — a presentation cheat sheet/report/article on the current[GenAI landscape](../sec_freeform/gen-ai-landscape-primer.md).
- 2025.09.12 Added support for PlantUML, as well as a current state of affairs before I move forward,[Diagram Accessibility in MkDocs Material: Current State &amp; Path Forward](../sec_accss/mkdocs-mermaid-plantuml-graphviz.md).
- 2025.08.27 Dropped JS injector approach in favor of accessibility mechanism in mermaid2; added local version of`mermaid.min.js`,[Breakthrough #1](../sec_accss/mermaid-beyond-reach.md).
- 2025.08.25 Reorganized content in subfolders for the new plugin,`awesome-pages` to semi-automate the navigation
- 2025.08.20 Enabled theme toggler; one last try to make accessibility metadata injector work; 'current state of affairs' as a bloggish piece of text,[Visuals for the Sightless](../sec_accss/visuals-for-the-sightless.md)
- 2025.08.19 Added some visuals to the AI-augmented Accessibility concept note; enabled some accessibility-related theme features
- 2025.08.18 Added a rough roadmap and a "concept note" based on the AI-augmented accessibility idea
- 2025.08.17 Added another first-ever; a 'technical daydream' on[AI-augmented accessibility](../sec_freeform/ai-accessibility.md)
- 2025.08.14 Site live via GitHub Pages
- 2025.08.13 Azure DevOps integration abandoned for technical reasons; GitHub Actions set up
- 2025.08.12 Added my first-ever 'technical memoir' on a pre-cloud-era Java project,[IBM-SanFrancisco](../sec_freeform/ibm-sanfran.md)
- 2025.08.08 Revival; attempting to connect with Azure DevOps; added a several 'contemporary' topics
- 2025.05.16 Attempt to revive the project and resume some GitHub activity
- 2021.01.24 Initial commit with a couple of glossaries for own use and to learn GitHub; shelved 2021.05.17
