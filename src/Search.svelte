<div class="max-w-xs md:max-w-md lg:max-w-xl flex-grow flex-shrink-0">
    <div class="relative h-10 my-4">
        <input placeholder="Search for a paper" type="search" on:keyup={handleInput} id="site-search" aria-label="Search for papers" class="focus:outline-0 border border-transparent focus:bg-white focus:border-gray-300 placeholder-gray-600 rounded-lg bg-gray-300 py-2 pr-4 pl-10 block w-full appearance-none leading-normal ds-input">
        <div class="pointer-events-none absolute inset-y-0 left-0 pl-4 flex items-center">
            <svg class="fill-current pointer-events-none text-gray-600 w-4 h-4" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
                <path d="M12.9 14.32a8 8 0 1 1 1.41-1.41l5.35 5.33-1.42 1.42-5.33-5.34zM8 14A6 6 0 1 0 8 2a6 6 0 0 0 0 12z"></path>
            </svg>
        </div>
    </div>
    {#if titles}
        <div class="flex flex-col">
            {#each titles as title (title.id)}
                <div on:click="{e => clickedPaper(title.id)}" class="bg-white rounded-lg shadow-xl p-4 my-2 hover:shadow-2xl hover:bg-gray-100">
                    {title.content}
                </div>
            {/each}
        </div>
    {/if}
</div>

<script>
    import 'whatwg-fetch';
    import { createEventDispatcher } from 'svelte';

	const dispatch = createEventDispatcher();
    var titles = [];

    var inputValue = "";
    var baseUrl = 'https://api.labs.cognitive.microsoft.com/academic/v1.0';
    $: intepretPath = '/interpret?query=' + inputValue  + '&complete=0&count=10&model=latest';

    function handleInput(event) {
        inputValue  = event.target.value;   

        if (event.key == "Enter" && inputValue) {            
            interpret().then(query => {
                evaluate(query)
            })
        }
    }

    function evaluate(query) {
        let attributes = ['Id', 'E'];
        let evaluatePath = "/evaluate?expr=" + query + "&attributes=" + attributes.join();

        console.log(query)
        fetchEvaluate(evaluatePath).then(evaluation => {
            evaluation.entities.forEach(function(element) {
                titles = [...titles, {id: element.Id, content: JSON.parse(element.E).DN}];
            });
            console.log(titles);
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

    function interpret() {
        return fetchInterpret()
            .then(res => {
                return res.interpretations[0].rules[0].output.value;
            })
            .catch(err => {
                console.error(err);
            });
    }

    function fetchInterpret() {
        return fetch(
            baseUrl + intepretPath,
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

    function clickedPaper(paperId) {
        dispatch('clickedPaper', {
            text: paperId
        });
    }
</script>