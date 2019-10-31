<div id="graphDiv"></div>

<script>
    import Viva from './vivagraph.js';
    
    export let originalPaperId = 40134741;

    const baseUrl = 'https://api.labs.cognitive.microsoft.com/academic/v1.0';

    let paperGraph = {papers: [], links: new Map()};

    loadGraph();

    async function loadGraph(){
        await evaluatePaper(`Id=${originalPaperId}`, ['Id', 'Ti', 'RId', 'AA.AuN'], 0);

        console.log('The papergraph');
        generateGraph(paperGraph.papers, paperGraph.links);
    }

    function evaluatePaper(query, attributes, recursionDepth) {
        let evaluatePath = "/evaluate?expr=" + query + "&attributes=" + attributes.join();

        fetchEvaluate(evaluatePath)
            .then(evaluation => {
                let paperEval = evaluation.entities[0];

                console.log(paperEval.Ti)
                let paper = {
                    id: paperEval.Id,
                    title: paperEval.Ti,
                    author: paperEval.AA.map(e => e.AuN).join(","),
                    cardWidth: (paperEval.Ti.length * 7),
                    cardHeight: 60
                }

                paperGraph.papers.push(paper);

                return { RId: paperEval.RId, paperEvalId: paperEval.Id };
            })
            .then(element => {
                if (recursionDepth < 1) {
                    element.RId.forEach(referencedPaperId => {
                        evaluatePaper(`Id=${referencedPaperId}`, attributes, recursionDepth + 1);
                        paperGraph.links.set(element.paperEvalId, referencedPaperId);
                    });
                } else {
                    console.log('finished')
                    return;
                }
            });
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
                    'Ocp-Apim-Subscription-Key': "554c1cf34356401ab6fc1dd31982f3ce",
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

        papers.forEach(paper => {
            let url = generateSVGStringFromPaper(paper);
            graph.addNode(paper.id, {url : url, cardWidth: paper.cardWidth, cardHeight: paper.cardHeight});
        });
        
        links.forEach(function(paper2, paper1, map) {
            graph.addLink(paper1, paper2);
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