{#if !graph}
    <div class="flex justify-center w-full pt-16">
      <h1 class="text-3xl font-light">Sending requests ({numberOfSentRequests})</h1>
    </div>
{/if}
<div id="graphDiv" on:mousemove={handleMouseMove} on:click={handeClick}>
    {#if tooltip.visible}
        <div class="tooltip z-10 absolute rounded shadow-xl bg-white px-4 py-2 max-w-4xl" style="left: {tooltip.x}px; top: {tooltip.y - 50}px">
            <p class="text-lg font-bold text-center capitalize overflow-hidden">{tooltip.title}</p>
            <p class="text-sm text-center capitalize overflow-hidden">{tooltip.author}</p>
        </div>
    {/if}
</div>

<script lang="javascript">
    import { onMount } from 'svelte';
    import Viva from './vivagraph.js';
    
    export let originalPaperId = 40134741;
    export let subscriptionKey;

    const baseUrl = 'https://api.labs.cognitive.microsoft.com/academic/v1.0';
    const maxNumberOfRequests = 200;
    const maxReferenceChildren = 10;
    const paperUrl = 'https://academic.microsoft.com/paper/';

    let numberOfSentRequests = 0;
    let graph;
    let tooltip = {
        visible: false,
        text: '',
        author: '',
        id: '',
        x: 0,
        y: 0
    }

    onMount(() => {
        loadGraph();
    });

    function loadGraph(){
        let paperGraph = { papers: new Map(), links: new Map() };
        
        if (localStorage.getItem(originalPaperId) === null) {
            evaluatePapers(paperGraph, `Id=${originalPaperId}`, 0).then(newPaperGraph => {
                localStorage.setItem(`${originalPaperId}`, JSON.stringify({ 
                    papers: Array.from(newPaperGraph.papers.entries()), 
                    links: Array.from(newPaperGraph.links.entries()) 
                }));
                console.log(newPaperGraph.papers.size)

                generateGraph(newPaperGraph.papers, newPaperGraph.links);
            });
        } else {
            let localPaperGraph = JSON.parse(localStorage.getItem(`${originalPaperId}`));
            
            paperGraph.papers = new Map(localPaperGraph.papers);
            paperGraph.links = new Map(localPaperGraph.links);

            generateGraph(paperGraph.papers, paperGraph.links);
        }
    }

    function evaluatePapers(paperGraph, query) {
        let evaluatePath = "/evaluate?expr=" + query + "&attributes=" + ['Id', 'Ti', 'AA.AuN', 'RId'].join();

        return fetchEvaluate(evaluatePath)
            .then(evaluation => {
                let paperEvals = [];

                evaluation.entities.forEach(paperEval => {
                    let paper = paperFromPaperEval(paperEval);
                    
                    paperGraph.papers.set(paper.id, paper);

                    paperEvals.push(paperEval);
                });


                return paperEvals;
            }).then(paperEvals => {
                if (paperEvals && paperEvals.length > 0) {
                    let requests = paperEvals.map(paperEval => {
                        return new Promise(resolve => {
                            if (paperEval.RId) {
                                paperEval.RId = paperEval.RId.slice(0, maxReferenceChildren);
                                
                                if (numberOfSentRequests + paperEval.RId.length > maxNumberOfRequests) {
                                    paperEval.RId = paperEval.RId.slice(0, maxNumberOfRequests - numberOfSentRequests);
                                }

                                if (paperEval.RId.length > 0) {                    
                                    paperEval.RId.forEach(referencedPaperId => {
                                        if (!paperGraph.links.has(paperEval.Id)) {
                                            paperGraph.links.set(paperEval.Id, [referencedPaperId]);
                                        } else {
                                            paperGraph.links.get(paperEval.Id).push(referencedPaperId);
                                        }
                                    });
                
                                    resolve(evaluateRIds(paperGraph, paperEval));
                                }
                            }

                            resolve(paperGraph);
                        });

                    });
                    
                    return Promise.all(requests);
                }

                return [paperGraph];
            }).then(newPaperGraphs => {
                newPaperGraphs.map(newPaperGraph => {
                    paperGraph.papers = new Map([...paperGraph.papers, ...newPaperGraph.papers]);
                    paperGraph.links = new Map([...paperGraph.links, ...newPaperGraph.links]);
                });

                return paperGraph;
            });
    }

    function evaluateRIds(paperGraph, paperEval) { 
        let ids = '';

        paperEval.RId.forEach(referencedPaperId => {
            ids += `Id=${referencedPaperId},`;
        });
        ids = ids.slice(0, -1);
                
        return new Promise(resolve => {
            resolve(evaluatePapers(paperGraph, `Or(${ids})`));
        });
    }

    function paperFromPaperEval(paperEval) {
        return {
            id: paperEval.Id,
            title: paperEval.Ti,
            author: paperEval.AA.map(e => e.AuN).join(", "),
            cardWidth: (paperEval.Ti.length * 7),
            cardHeight: 60
        }
    }

    function fetchEvaluate(evaluatePath) {
        numberOfSentRequests += 1;

        return fetch(
            baseUrl + evaluatePath,
            {
                method: 'GET',
                mode: 'cors',
                headers: new Headers({ 
                    'Host': 'api.labs.cognitive.microsoft.com',
                    'Connection': 'keep-alive',
                    'Content-Type': 'application/json',
                    'Ocp-Apim-Subscription-Key': subscriptionKey,
                }),
                cache: 'no-cache'
            }
        ).then(res => {
            if (res.status >= 400) {
                throw new Error("Bad response from server");
            }
            return res.json();
        })
        .catch(err => {
            console.error(err);
        });
    }

    function generateGraph(papers, links) {
        graph = Viva.Graph.graph();

        papers.forEach((paper, id, map) => {
            let imageUrl = 'test' //generateSVGStringFromPaper(paper);
            graph.addNode(paper.id, { imageUrl : imageUrl, cardWidth: paper.cardWidth, cardHeight: paper.cardHeight, title: paper.title, id: paper.id, author: paper.author });
        });
        
        links.forEach((papersReferenced, paperId, map) => {
            papersReferenced.map(referencedPaperId => {
                graph.addLink(paperId, referencedPaperId);
            });
        });

        let graphics = Viva.Graph.View.svgGraphics();
        
        graphics.node((node) => {
            return Viva.Graph
                .svg('rect')
                    .attr('width', 6)
                    .attr('height', 6)
                    .attr('id', node.data.id)
                    .attr('fill', '#1E90FF')
                    .attr('class', 'node')
                    .link(node.data.imageUrl); 
            })
            .placeNode((nodeUI, pos) => {
                nodeUI.attr('x', pos.x - 3).attr('y', pos.y - 3);
            });

        let renderer = Viva.Graph.View.renderer(graph, {
            container: document.getElementById('graphDiv'),
            graphics : graphics,
        });
        renderer.run();

        document.getElementById(originalPaperId).setAttribute('fill', '#012587');
    }

    function generateSVGStringFromPaper(paper) {
        let firstTitlePart = paper.title;
        let secondTitlePart = "";

        let splitLength = 20;

        if (paper.title.length > splitLength) {
            firstTitlePart = paper.title.substr(0, splitLength);
            secondTitlePart = paper.title.substr(splitLength, paper.title.length - splitLength);
        }

        let adjustScaling = paper.cardWidth < 200 ? true : false;
        paper.cardWidth = adjustScaling ? 200 : paper.cardWidth;

        let svg = ` <svg xmlns="http://www.w3.org/2000/svg">
                        <defs>
                            <filter id="shadow">
                                <feDropShadow dx="-1" dy="-1" stdDeviation="0.2"/>
                                <feDropShadow dx="1" dy="1" stdDeviation="0.2"/>
                                <feDropShadow dx="-1" dy="1" stdDeviation="0.2"/>
                                <feDropShadow dx="1" dy="-1" stdDeviation="0.2"/>
                            </filter>
                        </defs>
                        <rect x="5" y="5" width="${paper.cardWidth - 10}" height="${paper.cardHeight - 10}" rx="15" fill="white" filter="url(#shadow)"></rect>
                        <text x="25" y="25" textLength="${paper.cardWidth * 0.8}" lengthAdjust="spacingAndGlyphs" style="${adjustScaling ? '0.1rem' : '1rem'}">${paper.title}</text>
                        <text x="${paper.cardWidth / 2}" y="50" text-anchor="middle" style="font-size: 0.5rem;">${paper.author}</text>
                    </svg>`;

        let blob = new Blob([svg], {type: 'image/svg+xml'});
        let url = URL.createObjectURL(blob);
        let image = document.createElement('img');
        image.src = url;
        image.addEventListener('load', () => URL.revokeObjectURL(url), {once: true});

        return url;
    }

    function handleMouseMove(event) {
        if(!graph) return;

        const nodeId = getNodeIdFromDOM(event.target, 'node');
        if (!nodeId) {
            tooltip.visible = false;
            return;
        }

        clearHighlight();

        forAll(document.getElementById('graphDiv'), `path[data-from="${nodeId}"], path[data-to="${nodeId}"]`, addClass('hl'));

        const data = graph.getNode(nodeId).data;
        showTooltip(event.target, data);
    }

    function showTooltip(el, data) {
        const rect = el.getBoundingClientRect();
        tooltip.title = data.title;
        tooltip.author = data.author;
        tooltip.id = data.id;
        tooltip.x = rect.left + rect.width / 2;
        tooltip.y = rect.top - 42 / 2;
        tooltip.visible = true;
    }

    function forAll(scene, query, action) {
        (Array.from(scene.querySelectorAll(query))).forEach(action);
    }

    function addClass(className) {
        return el => el.classList.add(className);
    }

    function removeClass(className) {
        return el => el.classList.remove(className);
    }

    function getNodeIdFromDOM(el, type) {
        const isNode = el && el.classList.contains(type);
        if (!isNode) return;
        return el.getAttribute('id');
    }

    function clearHighlight() {
      forAll(document.getElementById('graphDiv'), '.hl', removeClass('hl'));
    }

    function handeClick(event) {
        if(!graph) return;

        const nodeId = getNodeIdFromDOM(event.target, 'node');
        if (!nodeId) return;

        window.location = paperUrl + tooltip.id;
    }
</script>

<style>
	:global(svg){
        @apply z-0;
        @apply relative;
		@apply h-screen;
		@apply w-screen;
    }
	:global(canvas){
		@apply h-screen;
		@apply w-screen;
    }

    :global(path.hl) {
        stroke: RGB(51, 182, 121);
        stroke-width: 2px;
    }

    :global(.tooltip) {
        pointer-events: none;
        transform: translate(-50%, 0);
        white-space: nowrap;
    }
</style>