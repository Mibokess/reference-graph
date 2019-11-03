{#if !graphRendered && maxNumberOfRequests != 20}
    <div class="flex justify-center w-full pt-16">
      <h1 class="text-3xl font-light">Sending requests ({numberOfSentRequests})</h1>
    </div>
{/if}
<div id="graphDiv" on:mousemove={handleMouseMove} on:click={handleClick}>
    {#if tooltip.visible}
        <div class="tooltip z-10 absolute rounded shadow-xl bg-white px-4 py-2 max-w-4xl" style="left: {tooltip.x}px; top: {tooltip.y - 50}px">
            <p class="text-lg font-bold text-center capitalize overflow-hidden">{tooltip.title}</p>
            <p class="text-sm text-center capitalize overflow-hidden">{tooltip.author}</p>
        </div>
    {/if}
</div>

<script lang="javascript">
    import { onMount } from 'svelte';
    import { afterUpdate } from 'svelte';
    import Viva from './vivagraph.js';
    
    export let paperId;
    export let currentPaperId = -1;
    export let subscriptionKey;
    export let maxNumberOfRequests;

    const baseUrl = 'https://api.labs.cognitive.microsoft.com/academic/v1.0';
    const maxReferenceChildren = 10;
    const maxRecursionDepth = 5;
    const paperUrl = 'https://academic.microsoft.com/paper/';

    let numberOfSentRequests = 0;
    let graph;
    let renderer;
    let graphRendered = false;
    let tooltip = {
        visible: false,
        text: '',
        author: '',
        id: '',
        x: 0,
        y: 0
    }

    let hasMouseDown = false;

    onMount(() => {
        localStorage.setItem(paperId, heliosgraphJSON);

        loadGraph();
    });

    afterUpdate(() => {
        if (graphRendered && currentPaperId != paperId) {
            console.log('after')
            loadGraph();
            currentPaperId = paperId;
        }
    });

    function loadGraph(){
        if (graphRendered) {
            graph.clear();
            graphRendered = false;
        }

        let paperGraph = { papers: new Map(), links: new Map() };
        
        if (localStorage.getItem(paperId) === null) {
            evaluatePapers(paperGraph, `Id=${paperId}`, 0).then(newPaperGraph => {
                localStorage.setItem(`${paperId}`, JSON.stringify({ 
                    papers: Array.from(newPaperGraph.papers.entries()), 
                    links: Array.from(newPaperGraph.links.entries()) 
                }));
                console.log(newPaperGraph.papers.size)

                generateGraph(newPaperGraph.papers, newPaperGraph.links);
            });
        } else {
            let localPaperGraph = JSON.parse(localStorage.getItem(`${paperId}`));
            
            paperGraph.papers = new Map(localPaperGraph.papers);
            paperGraph.links = new Map(localPaperGraph.links);

            generateGraph(paperGraph.papers, paperGraph.links);
        }
    }

    function evaluatePapers(paperGraph, query, recursionDepth) {
        let evaluatePath = "/evaluate?expr=" + query + "&attributes=" + ['Id', 'Ti', 'AA.AuN', 'RId'].join();

        return fetchEvaluate(evaluatePath)
            .then(evaluation => {
                let paperEvals = [];

                evaluation.entities.forEach(paperEval => {
                    if (!paperGraph.papers.has(paperEval.Id)) {
                        let paper = paperFromPaperEval(paperEval);
                        
                        paperGraph.papers.set(paper.id, paper);

                        paperEvals.push(paperEval);
                    }
                });


                return paperEvals;
            }).then(paperEvals => {
                if (paperEvals && paperEvals.length > 0 && recursionDepth < maxRecursionDepth) {
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
                
                                    resolve(evaluateRIds(paperGraph, paperEval, recursionDepth));
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

    function evaluateRIds(paperGraph, paperEval, recursionDepth) { 
        let ids = '';

        paperEval.RId.forEach(referencedPaperId => {
            ids += `Id=${referencedPaperId},`;
        });
        ids = ids.slice(0, -1);
                
        return new Promise(resolve => {
            resolve(evaluatePapers(paperGraph, `Or(${ids})`, recursionDepth + 1));
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
        if (!graph) {
            graph = Viva.Graph.graph();
        }

        papers.forEach((paper, id, map) => {
            graph.addNode(paper.id, { 
                cardWidth: paper.cardWidth, 
                cardHeight: paper.cardHeight, 
                title: paper.title, 
                id: paper.id, 
                author: paper.author, 
                timesReferenced: timesReferenced(paper.id, links)
            });
        });

        links.forEach((papersReferenced, paperId, map) => {
            papersReferenced.map(referencedPaperId => {
                graph.addLink(paperId, referencedPaperId);
            });
        });
        
        let root = graph.getNode(paperId);
        root.isPinned = true;

        let graphics = Viva.Graph.View.svgGraphics();
        graphics.node(node => {
            return Viva.Graph
                .svg('rect')
                    .attr('width', 6 + Math.sqrt(node.data.timesReferenced * 20))
                    .attr('height', 6 + Math.sqrt(node.data.timesReferenced * 20))
                    .attr('id', node.data.id)
                    .attr('fill', '#1E90FF')
                    .attr('class', 'node')
            })
            .placeNode((nodeUI, pos) => {
                nodeUI.attr('x', pos.x - 3).attr('y', pos.y - 3);
            });

        if (!graphRendered && !renderer) {
            renderer = Viva.Graph.View.renderer(graph, {
                container: document.getElementById('graphDiv'),
                graphics : graphics,
                interactive: 'drag scroll'
            });

            renderer.run();
        } else {
            console.log('rerender')
            renderer.rerender();
        }

        if (document.getElementById(paperId) != null) document.getElementById(paperId).setAttribute('fill', '#012587');

        graphRendered = true;
        numberOfSentRequests = 0;
    }

    function timesReferenced(paperId, links) {
        let counter = 0;

        (Array.from(links.values())).forEach(references => {
            if (references) {
                references.map(reference => {
                    if (paperId == reference) counter++;
                });
            }
        });

        return counter;
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

    function handleClick(event) {
        if(!graph) return;

        const nodeId = getNodeIdFromDOM(event.target, 'node');
        if (!nodeId) return;

        window.open(paperUrl + tooltip.id);
    }

    let heliosgraphJSON = '{"papers":[[40134741,{"id":40134741,"title":"helios web based open audit voting","author":"ben adida","cardWidth":238,"cardHeight":60}],[1589034595,{"id":1589034595,"title":"how to prove yourself practical solutions to identification and signature problems","author":"amos fiat, adi shamir","cardWidth":574,"cardHeight":60}],[1595293097,{"id":1595293097,"title":"the secure remote password protocol","author":"thomas d wu","cardWidth":245,"cardHeight":60}],[2007216019,{"id":2007216019,"title":"coercion resistant electronic elections","author":"ari juels, dario catalano, markus jakobsson","cardWidth":273,"cardHeight":60}],[2153193245,{"id":2153193245,"title":"wallet databases with observers","author":"david chaum, torben p pedersen","cardWidth":217,"cardHeight":60}],[2101770573,{"id":2101770573,"title":"a verifiable secret shuffle and its application to e voting","author":"c andrew neff","cardWidth":413,"cardHeight":60}],[2108079590,{"id":2108079590,"title":"advances in cryptographic voting systems","author":"ronald l rivest, ben adida","cardWidth":280,"cardHeight":60}],[1572349943,{"id":1572349943,"title":"simple verifiable elections","author":"josh benaloh","cardWidth":189,"cardHeight":60}],[1593702530,{"id":1593702530,"title":"receipt free mix type voting scheme a practical solution to the implementation of a voting booth","author":"kazue sako, joe kilian","cardWidth":672,"cardHeight":60}],[634465965,{"id":634465965,"title":"secure electronic voting","author":"dimitris gritzalis","cardWidth":168,"cardHeight":60}],[2147591189,{"id":2147591189,"title":"ballot casting assurance via voter initiated poll station auditing","author":"josh benaloh","cardWidth":462,"cardHeight":60}],[2151413173,{"id":2151413173,"title":"a digital signature scheme secure against adaptive chosen message attacks","author":"shafi goldwasser, silvio micali, ronald l rivest","cardWidth":511,"cardHeight":60}],[1535861450,{"id":1535861450,"title":"untraceable electronic cash","author":"david chaum, amos fiat, moni naor","cardWidth":189,"cardHeight":60}],[2010553858,{"id":2010553858,"title":"blind signature systems","author":"david chaum","cardWidth":161,"cardHeight":60}],[2142968417,{"id":2142968417,"title":"witness indistinguishable and witness hiding protocols","author":"uri feige, adi shamir","cardWidth":378,"cardHeight":60}],[2915470408,{"id":2915470408,"title":"advances in cryptology proceedings of crypto 84","author":"george robert blakley, david chaum","cardWidth":329,"cardHeight":60}],[1569083856,{"id":1569083856,"title":"identity based cryptosystems and signature schemes","author":"adi shamir","cardWidth":350,"cardHeight":60}],[2144752539,{"id":2144752539,"title":"the knowledge complexity of interactive proof systems","author":"shafi goldwasser, silvio micali, charles rackoff","cardWidth":371,"cardHeight":60}],[2117362057,{"id":2117362057,"title":"how to construct random functions","author":"oded goldreich, shafi goldwasser, silvio micali","cardWidth":231,"cardHeight":60}],[2148373594,{"id":2148373594,"title":"proofs that yield nothing but their validity and a methodology of cryptographic protocol design","author":"oded goldreich, silvio micali, avi wigderson","cardWidth":665,"cardHeight":60}],[1504232224,{"id":1504232224,"title":"a secure protocol for rhe oblivious transfer","author":"michael j fischer, silvio micali, charles rackoff","cardWidth":308,"cardHeight":60}],[1225131154,{"id":1225131154,"title":"proofs that yield nothing but the validity of the assertion and a methodology of cryptographic pro","author":"oded goldreich, silvio micali, avi wigderson","cardWidth":686,"cardHeight":60}],[1952573265,{"id":1952573265,"title":"how to construct randolli functions","author":"oded goldreich, shafi goldwasser, silvio micali","cardWidth":245,"cardHeight":60}],[2156186849,{"id":2156186849,"title":"new directions in cryptography","author":"whitfield diffie, martin e hellman","cardWidth":210,"cardHeight":60}],[2157604883,{"id":2157604883,"title":"encrypted key exchange password based protocols secure against dictionary attacks","author":"steven m bellovin, michael merritt","cardWidth":567,"cardHeight":60}],[2133432179,{"id":2133432179,"title":"strong password only authenticated key exchange","author":"david p jablon","cardWidth":329,"cardHeight":60}],[2167006959,{"id":2167006959,"title":"protecting poorly chosen secrets from guessing attacks","author":"li gong, m a lomas, roger m needham, jerome h saltzer","cardWidth":378,"cardHeight":60}],[2080302839,{"id":2080302839,"title":"message recovery for signature schemes based on the discrete logarithm problem","author":"kaisa nyberg, rainer a rueppel","cardWidth":546,"cardHeight":60}],[2091020476,{"id":2091020476,"title":"augmented encrypted key exchange a password based protocol secure against dictionary attacks and password file compromise","author":"steven m bellovin, michael merritt","cardWidth":847,"cardHeight":60}],[2245636100,{"id":2245636100,"title":"on internet authentication","author":"n haller, r atkinson","cardWidth":182,"cardHeight":60}],[2043361728,{"id":2043361728,"title":"the unix system unix operating system security","author":"f t grampp, r h morris","cardWidth":322,"cardHeight":60}],[2043926312,{"id":2043926312,"title":"elliptic curve cryptosystems and their implementation","author":"alfred menezes, scott a vanstone","cardWidth":371,"cardHeight":60}],[74111940,{"id":74111940,"title":"unix password security","author":"ken thompson","cardWidth":154,"cardHeight":60}],[1660562555,{"id":1660562555,"title":"handbook of applied cryptography","author":"alfred menezes, scott a vanstone, paul c van oorschot","cardWidth":224,"cardHeight":60}],[1655958391,{"id":1655958391,"title":"tor the second generation onion router","author":"roger dingledine, nick mathewson, paul syverson","cardWidth":266,"cardHeight":60}],[1996360405,{"id":1996360405,"title":"a method for obtaining digital signatures and public key cryptosystems","author":"ronald l rivest, adi shamir, leonard m adleman","cardWidth":490,"cardHeight":60}],[2132172731,{"id":2132172731,"title":"public key cryptosystems based on composite degree residuosity classes","author":"pascal paillier","cardWidth":490,"cardHeight":60}],[1603565383,{"id":1603565383,"title":"captcha using hard ai problems for security","author":"luis von ahn, manuel blum, nicholas hopper, john langford","cardWidth":301,"cardHeight":60}],[2108834246,{"id":2108834246,"title":"a public key cryptosystem and a signature scheme based on discrete logarithms","author":"taher elgamal","cardWidth":539,"cardHeight":60}],[2913419753,{"id":2913419753,"title":"visual cryptography","author":"moni naor, a shamir","cardWidth":133,"cardHeight":60}],[1798609567,{"id":1798609567,"title":"evaluating 2 dnf formulas on ciphertexts","author":"dan boneh, eujin goh, kobbi nissim","cardWidth":280,"cardHeight":60}],[1499934958,{"id":1499934958,"title":"universally composable security a new paradigm for cryptographic protocols","author":"ran canetti","cardWidth":518,"cardHeight":60}],[2052267638,{"id":2052267638,"title":"random oracles are practical a paradigm for designing efficient protocols","author":"mihir bellare, phillip rogaway","cardWidth":511,"cardHeight":60}],[2141420453,{"id":2141420453,"title":"how to share a secret","author":"adi shamir","cardWidth":147,"cardHeight":60}],[2095708839,{"id":2095708839,"title":"efficient signature generation by smart cards","author":"clauspeter schnorr","cardWidth":315,"cardHeight":60}],[2101803085,{"id":2101803085,"title":"a practical public key cryptosystem provably secure against adaptive chosen ciphertext attack","author":"ronald cramer, victor shoup","cardWidth":651,"cardHeight":60}],[2123510989,{"id":2123510989,"title":"analysis of an electronic voting system","author":"tadayoshi kohno, adam stubblefield, aviel d rubin, dan s wallach","cardWidth":273,"cardHeight":60}],[2103647628,{"id":2103647628,"title":"untraceable electronic mail return addresses and digital pseudonyms","author":"david chaum","cardWidth":469,"cardHeight":60}],[2165210192,{"id":2165210192,"title":"an efficient system for non transferable anonymous credentials with optional anonymity revocation","author":"jan camenisch, anna lysyanskaya","cardWidth":679,"cardHeight":60}],[2137739383,{"id":2137739383,"title":"secret ballot receipts true voter verifiable elections","author":"david chaum","cardWidth":378,"cardHeight":60}],[146445700,{"id":146445700,"title":"handbook of applied cryptography boca raton","author":"alfred john menezes, paul c van oorschot, scott a vanstone","cardWidth":301,"cardHeight":60}],[2138784757,{"id":2138784757,"title":"a secure and optimally efficient multi authority election scheme","author":"ronald cramer, rosario gennaro, berry schoenmakers","cardWidth":448,"cardHeight":60}],[2112138431,{"id":2112138431,"title":"practical threshold signatures","author":"victor shoup","cardWidth":210,"cardHeight":60}],[1932825448,{"id":1932825448,"title":"a practical voter verifiable election scheme","author":"david chaum, peter y a ryan, steve schneider","cardWidth":308,"cardHeight":60}],[115629558,{"id":115629558,"title":"constructing digital signatures from a one way function","author":"leslie lamport","cardWidth":385,"cardHeight":60}],[1987667503,{"id":1987667503,"title":"a randomized protocol for signing contracts","author":"shimon even, oded goldreich, a lempel","cardWidth":301,"cardHeight":60}],[1964053266,{"id":1964053266,"title":"a fast monte carlo test for primality","author":"robert solovay, volker strassen","cardWidth":259,"cardHeight":60}],[2106287110,{"id":2106287110,"title":"hiding information and signatures in trapdoor knapsacks","author":"ralph c merkle, martin e hellman","cardWidth":385,"cardHeight":60}],[2151433956,{"id":2151433956,"title":"minimum disclosure proofs of knowledge","author":"gilles brassard, david chaum, claude crepeau","cardWidth":266,"cardHeight":60}],[1479744988,{"id":1479744988,"title":"a practical secret voting scheme for large scale elections","author":"atsushi fujioka, tatsuaki okamoto, kazuo ohta","cardWidth":406,"cardHeight":60}],[2105564033,{"id":2105564033,"title":"verifiable secret ballot elections","author":"josh benaloh","cardWidth":238,"cardHeight":60}],[2003593381,{"id":2003593381,"title":"receipt free secret ballot elections extended abstract","author":"josh benaloh, dwight tuinstra","cardWidth":378,"cardHeight":60}],[1570270086,{"id":1570270086,"title":"receipt free secret ballot elections","author":"josh benaloh","cardWidth":252,"cardHeight":60}],[1839300275,{"id":1839300275,"title":"secure voting using partially compatible homomorphisms","author":"kazue sako, joe kilian","cardWidth":378,"cardHeight":60}],[2092657829,{"id":2092657829,"title":"distributing the power of a government to enhance the privacy of voters","author":"josh benaloh, moti yung","cardWidth":497,"cardHeight":60}],[2287961307,{"id":2287961307,"title":"breaking an efficient anonymous channel","author":"birgit pfitzmann","cardWidth":273,"cardHeight":60}],[1494929562,{"id":1494929562,"title":"making mix nets robust for electronic voting by randomized partial checking","author":"markus jakobsson, ari juels, ronald l rivest","cardWidth":525,"cardHeight":60}],[1878594160,{"id":1878594160,"title":"an efficient scheme for proving a shuffle","author":"jun furukawa, kazue sako","cardWidth":287,"cardHeight":60}],[2031618446,{"id":2031618446,"title":"security without identification transaction systems to make big brother obsolete","author":"david chaum","cardWidth":560,"cardHeight":60}],[2087811006,{"id":2087811006,"title":"the dining cryptographers problem unconditional sender and recipient untraceability","author":"david chaum","cardWidth":581,"cardHeight":60}],[2074594718,{"id":2074594718,"title":"one way functions are necessary and sufficient for secure signatures","author":"j rompel","cardWidth":476,"cardHeight":60}],[1601001795,{"id":1601001795,"title":"blind signatures for untraceable payments","author":"devid l chaum","cardWidth":287,"cardHeight":60}],[2165992167,{"id":2165992167,"title":"cryptographic communications system and method","author":"ronald l rivest, adi shamir, leonard m adleman","cardWidth":322,"cardHeight":60}],[2003647955,{"id":2003647955,"title":"multiuser cryptographic techniques","author":"whitfield diffie, martin e hellman","cardWidth":238,"cardHeight":60}],[1808229546,{"id":1808229546,"title":"cryptographic apparatus and method","author":"martin e hellman, bailey w diffie, ralph c merkle","cardWidth":238,"cardHeight":60}],[2097158416,{"id":2097158416,"title":"public key cryptographic apparatus and method","author":"martin e hellman, ralph c merkle","cardWidth":315,"cardHeight":60}],[1870703228,{"id":1870703228,"title":"method and apparatus providing registered mail features in an electronic communication system","author":"christian muellerschloer","cardWidth":651,"cardHeight":60}],[1970606468,{"id":1970606468,"title":"the knowledge complexity of interactive proof systems","author":"shafi goldwasser, silvio micali, charles rackoff","cardWidth":371,"cardHeight":60}],[1979215153,{"id":1979215153,"title":"zero knowledge proofs of identity","author":"uriel feige, amos fiat, adi shamir","cardWidth":231,"cardHeight":60}],[2752853835,{"id":2752853835,"title":"the art of computer programming","author":"donald ervin knuth","cardWidth":217,"cardHeight":60}],[2061106457,{"id":2061106457,"title":"theory and application of trapdoor functions","author":"andrew chichih yao","cardWidth":308,"cardHeight":60}],[2036265926,{"id":2036265926,"title":"the complexity of theorem proving procedures","author":"stephen a cook","cardWidth":308,"cardHeight":60}],[2088135982,{"id":2088135982,"title":"knowledge and common knowledge in a distributed environment","author":"joseph y halpern, yoram moses","cardWidth":413,"cardHeight":60}],[1864248001,{"id":1864248001,"title":"a randomized protocol for signing contracts","author":"shimon even, oded goldreich, a lempel","cardWidth":301,"cardHeight":60}],[2096390054,{"id":2096390054,"title":"trading group theory for randomness","author":"laszlo babai","cardWidth":245,"cardHeight":60}],[2019851160,{"id":2019851160,"title":"games against nature","author":"christos h papadimitriou","cardWidth":140,"cardHeight":60}],[1655990431,{"id":1655990431,"title":"the design and analysis of computer algorithms","author":"alfred v aho, john e hopcroft","cardWidth":322,"cardHeight":60}]],"links":[[40134741,[1589034595,1595293097,2007216019,2153193245,2101770573,2108079590,1572349943,1593702530,634465965,2147591189]],[1589034595,[1569083856,2144752539,2117362057,2148373594,1504232224,1225131154,1952573265]],[1595293097,[2156186849,2157604883,2133432179,2167006959,2080302839,2091020476,2245636100,2043361728,2043926312,74111940]],[2007216019,[2132172731,2052267638,2108834246,2141420453,2095708839,2101803085,2123510989,2103647628,2165210192,2137739383]],[2153193245,[1589034595,2151413173,1535861450,2010553858,2142968417,2915470408]],[2101770573,[1660562555,2156186849,146445700,2108834246,2141420453,1589034595,2095708839,2103647628,2138784757,2153193245]],[2108079590,[1660562555,1655958391,1996360405,2156186849,2132172731,1603565383,2108834246,2913419753,1798609567,1499934958]],[1572349943,[1996360405,2156186849,2108834246,2141420453,1589034595,2144752539,2103647628,2137739383,2112138431,1932825448]],[1593702530,[1589034595,2103647628,2151433956,1479744988,2105564033,2003593381,1570270086,1839300275,2092657829,2287961307]],[2147591189,[2103647628,2137739383,1932825448,2138784757,1494929562,2101770573,1878594160,1572349943,1593702530,2105564033]],[2151413173,[1996360405,2156186849,2108834246,2144752539,2117362057,115629558,1987667503,1964053266,2106287110]],[1535861450,[1996360405,2144752539,2103647628,2031618446,2148373594,2087811006,2151433956,2074594718]],[2010553858,[1996360405,1601001795,2165992167,2003647955,1808229546,2097158416,1870703228]],[2142968417,[1970606468,2151413173,2117362057,2148373594,1979215153,2151433956]],[2144752539,[2036265926,2088135982,1864248001,2096390054,2019851160]],[2117362057,[1996360405,2156186849,2752853835,2061106457]],[2148373594,[2141420453,2144752539,2036265926]],[1952573265,[1996360405,2156186849]],[2156186849,[1655990431]]]}';
</script>

<style>
	:global(svg){
        @apply z-10;
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