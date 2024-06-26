<script lang="ts">
    import { currentHeading, Checkboxes } from '@components/store';
    import * as icons from 'file-icons-js';
    import 'file-icons-js/css/style.css';
    import { onMount } from 'svelte';
    import CopyButton from '@components/CopyButton.svelte';
    import { Confetti } from 'svelte-confetti';

    export let headings:
        | { text: string; slug: string; depth: number; fa_icon?: string; checkboxes?: [] }[]
        | undefined = [];

    let mermaid;

    onMount(async () => {
        async function renderDiagrams(graphs) {
            mermaid = await import('mermaid').then((module) => module.default);
            mermaid.initialize({
                startOnLoad: false,
                fontFamily: 'var(--sans-font)',
                // @ts-ignore This works, but TS expects a enum for some reason
                theme: document.documentElement.getAttribute('data-bs-theme') === 'dark' ? 'dark' : 'neutral',
            });

            for (const graph of graphs) {
                const content = graph.getAttribute('data-content');
                if (!content) continue;
                let svg = document.createElement('svg');
                const id = (svg.id = 'mermaid-' + Math.round(Math.random() * 100000));
                graph.appendChild(svg);
                mermaid.render(id, content).then((result) => {
                    graph.innerHTML = result.svg;
                });
            }
        }
        const graphs = document.getElementsByClassName('mermaid');
        if (document.getElementsByClassName('mermaid').length > 0) {
            renderDiagrams(graphs);
            window.addEventListener('theme-changed', (e) => {
                renderDiagrams(graphs);
            });
        }

        // find current heading in viewport with IntersectionObserver
        const observer = new IntersectionObserver(
            (entries) => {
                entries.forEach((entry) => {
                    if (entry.isIntersecting) {
                        currentHeading.set(entry.target.id);
                    }
                });
            },
            {
                rootMargin: '0px 0px -92% 0px',
            },
        );
        // set current heading to last heading when at bottom of page
        document.addEventListener('scrollend', () => {
            if (window.innerHeight + window.scrollY >= document.body.offsetHeight) {
                currentHeading.set(headings[headings.length - 1].slug);
            }
        });

        const handleCheckboxes = (checkbox) => {
            // handle nested checkboxes under current checkbox
            const childChecks = checkbox.closest('li')?.querySelectorAll('input[type="checkbox"]');
            if (childChecks) {
                childChecks.forEach((c) => {
                    c.checked = checkbox.checked;
                    updateHeadingsCheckbox(c);
                });
            }
            // handle parent checkbox
            const parentCheckbox = checkbox
                .closest('ul')
                ?.previousElementSibling?.querySelector('input[type="checkbox"]');
            if (parentCheckbox) {
                const siblingChecks = checkbox.closest('ul')?.querySelectorAll('input[type="checkbox"]');
                if (siblingChecks && Array.from(siblingChecks).every((c) => c.checked)) {
                    parentCheckbox.checked = true;
                    updateHeadingsCheckbox(parentCheckbox);
                } else {
                    parentCheckbox.checked = false;
                    updateHeadingsCheckbox(parentCheckbox);
                }
            }

            // update checkboxes store
            Checkboxes.set(
                headings
                    ?.map((h) => h.checkboxes)
                    .map((c) => c)
                    .flat()
                    .filter((checkbox) => checkbox !== undefined),
            );
        };
        const updateHeadingsCheckbox = (c) => {
            const heading = headings?.find((h) => h.checkboxes?.find((ch) => ch.id === c.id));
            if (heading) {
                const checkbox = heading.checkboxes?.find((ch) => ch.id === c.id);
                if (checkbox) {
                    checkbox.checked = c.checked;
                }
            }
        };

        const markAllCheckboxes = (checkbox) => {
            // add is-valid class to all checked checkboxes if their id starts with the same string (everything except the number at the end)
            const headingID = checkbox.id.replace('checkbox-', '').replace(/-[0-9]+$/g, '');
            const currentHeading = headings?.find((h) => h.slug === headingID);
            const allChecked = currentHeading?.checkboxes?.every((c) => c.checked);
            if (allChecked) {
                document.querySelectorAll(`input[id^="checkbox-${headingID}"]`).forEach((c) => {
                    c.classList.add('is-valid');
                });
            } else {
                document.querySelectorAll(`input[id^="checkbox-${headingID}"]`).forEach((c) => {
                    c.classList.remove('is-valid');
                });
            }
        };
        if (headings?.some((h) => h.checkboxes)) {
            // add Confetti component to last unchecked checkbox
            const addConfetti = (checkbox) => {
                const confetti = new Confetti({ target: checkbox.parentElement, props: { rounded: true } });
            };

            // add event listener to add Confetti component to last unchecked checkbox
            document.addEventListener('change', (e) => {
                const checkbox = e.target as HTMLInputElement;
                if (checkbox.type === 'checkbox' && headings?.every((h) => h.checkboxes?.every((c) => c.checked))) {
                    addConfetti(checkbox);
                }
            });
        }

        headings?.forEach((heading) => {
            const element = document.querySelector('#' + heading.slug);
            if (element) {
                observer.observe(element);
            }
            if (heading.checkboxes) {
                heading.checkboxes.forEach((checkbox) => {
                    const checkboxElement = document.getElementById(checkbox.id);
                    // check checkboxes based on checkboxes store
                    if (checkboxElement) {
                        // set checkbox to checked if it is in the Checkboxes store
                        checkboxElement.checked = $Checkboxes.find((c) => c.id === checkbox.id)?.checked || false;
                        // set checkboxes in headings to the state in the Checkboxes store
                        updateHeadingsCheckbox(checkboxElement);
                        // add event listener to update Checkboxes store when checkbox is checked
                        checkboxElement.addEventListener('change', () => {
                            handleCheckboxes(checkboxElement);
                            markAllCheckboxes(checkbox);
                        });
                    }
                });

                markAllCheckboxes(document.getElementById(heading.checkboxes[0].id));
            }
        });
        // Add "Copy code" button in code blocks
        const copyButtonLabel = "<i class='fa-regular fa-clipboard'></i>";
        const copiedButtonLabel = `<span class='font-sans-serif'>Copied </span><i class='fa-regular fa-clipboard-check'></i>`;
        document
            .querySelectorAll("figure[data-rehype-pretty-code-figure] pre:not([data-language='console'])")
            .forEach((block) => {
                block.classList.add('position-relative');
                const copyText = block.querySelector('code')?.innerText;
                if (copyText) {
                    // check if block has only one child, i.e. is a single line code block, so we need less top and bottom margin for button
                    const SingleLine = block.childElementCount === 1 ? 'single-line' : '';
                    new CopyButton({
                        target: block, // Specify the target element for the Svelte component
                        props: {
                            text: copyText,
                            label: copyButtonLabel,
                            copiedLabel: copiedButtonLabel,
                            classes:
                                SingleLine +
                                ' copy-code-button btn btn-sm btn-outline-secondary position-absolute top-0 end-0',
                            copiedClasses:
                                SingleLine +
                                ' copy-code-button btn btn-sm btn-outline-success position-absolute top-0 end-0',
                        },
                    });
                }
            });
        // Add file icon to code block titles
        document.querySelectorAll('[data-rehype-pretty-code-title]').forEach((block) => {
            const title = block.textContent;
            const fileIcon = icons.getClass(title);
            let icon: HTMLElement;
            if (fileIcon) {
                icon = document.createElement('span');
                icon.classList.add('ms-1', 'me-2', fileIcon);
            } else {
                icon = document.createElement('i');
                icon.classList.add('fa-regular', 'fa-file-code', 'ms-1', 'me-2');
            }
            block.prepend(icon);
        });
    });
</script>

<div>
    <slot />
</div>
