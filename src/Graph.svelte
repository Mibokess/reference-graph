<div class="h-screen w-screen" id="graphDiv" on:click={loadGraph}></div>

<script>
    import Viva from './vivagraph.js';
    
    export let originalPaperId;
    export let subscriptionKey;

    const baseUrl = 'https://api.labs.cognitive.microsoft.com/academic/v1.0';

    function loadGraph(){
        let paperGraph = { papers: new Map(), links: new Map() };

        evaluatePaper(paperGraph, `Id=${originalPaperId}`, 0).then(newPaperGraph => {
            console.log('The papergraph');
            console.log(newPaperGraph);
            console.log(newPaperGraph.papers.length)
            generateGraph(newPaperGraph.papers, newPaperGraph.links);
        });
    }

    function evaluatePaper(paperGraph, query, recursionDepth) {
        console.log('enter evaluatePaper')
        let evaluatePath = "/evaluate?expr=" + query + "&attributes=" + ['Id', 'Ti', 'AA.AuN', 'RId'].join();
        
        Promise.resolve(fetchEvaluate(evaluatePath)
            .then(evaluation => {
                let paperEval = evaluation.entities[0];
                let paper = paperFromPaperEval(paperEval);
                
                paperGraph.papers.set(paper.id, paper);

                return paperEval;
            }).then((paperEval) => {
                Promise.resolve(evaluateRIds(paperGraph, recursionDepth, paperEval).then(newPaperGraph => {
                    if (paperEval.Id == 40134741) console.log(paperGraph);
                    return newPaperGraph;
                }));
            }));
    }

    function evaluateRIds(paperGraph, recursionDepth, paperEval) { 
        console.log('enter evaluateRIds')
        return new Promise((resolve) => {
            if (recursionDepth < 1 && paperEval.RId) { 
                paperEval.RId.forEach(referencedPaperId => {
                    paperGraph.links.set(paperEval.Id, referencedPaperId);
                });

                let newPaperGraphs = []
                let requests = paperEval.RId.map(referencedPaperId => {
                    return new Promise(resolve => {
                        evaluatePaper(paperGraph, `Id=${referencedPaperId}`, recursionDepth + 1).then(newPaperGraph => {
                            newPaperGraphs.push(newPaperGraph);
                        });
                    });
                });

                Promise.all(requests).then(() => {
                    newPaperGraphs.reduce(function(paperGraph, newPaperGraph) {
                        paperGraph.papers = new Map([...paperGraph.papers, ...newPaperGraph.papers]);
                        paperGraph.links = new Map([...paperGraph.links, ...newPaperGraph.links]);
                    });

                    return paperGraph;
                });
            } else {
                return paperGraph;
            }
        });
    }

    function paperFromPaperEval(paperEval) {
        return {
            id: paperEval.Id,
            title: paperEval.Ti,
            author: paperEval.AA.map(e => e.AuN).join(","),
            cardWidth: (paperEval.Ti.length * 7),
            cardHeight: 60
        }
    }

    function fetchEvaluate(evaluatePath) {
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
        var graph = Viva.Graph.graph();

        papers.forEach((id, paper, map) => {
            console.log(paper)
            let url = generateSVGStringFromPaper(paper);
            graph.addNode(paper.id, { url : url, cardWidth: paper.cardWidth, cardHeight: paper.cardHeight });
        });
        
        links.forEach(function(paper2Id, paper1Id, map) {
            graph.addLink(paper1Id, paper2Id);
        });

        var graphics = Viva.Graph.View.svgGraphics();
        graphics.node(function(node) {
            return Viva.Graph
                .svg('image')
                    .attr('width', node.data.cardWidth)
                    .attr('height', node.data.cardHeight)
                    .link(node.data.url); 
            })
            .placeNode(function(nodeUI, pos){
                // Shift image to let links go to the center:
                nodeUI.attr('x', pos.x - 12).attr('y', pos.y - 12);
            });

        var renderer = Viva.Graph.View.renderer(graph, {
                graphics : graphics
            });
        renderer.run();
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
</script>

<style>
	:global(svg){
		@apply h-screen;
		@apply w-screen;
    }
</style>