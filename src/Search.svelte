<div class="max-w-xs md:max-w-md lg:max-w-xl flex-grow flex-shrink-0">
    <div class="relative h-10">
        <input placeholder="Search for a paper" type="search" on:keyup={handleInput} id="site-search" aria-label="Search for papers" class="focus:outline-0 border border-transparent focus:bg-white focus:border-gray-300 placeholder-gray-600 rounded-lg bg-gray-200 py-2 pr-4 pl-10 block w-full appearance-none leading-normal ds-input">
        <div class="pointer-events-none absolute inset-y-0 left-0 pl-4 flex items-center">
            <svg class="fill-current pointer-events-none text-gray-600 w-4 h-4" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
                <path d="M12.9 14.32a8 8 0 1 1 1.41-1.41l5.35 5.33-1.42 1.42-5.33-5.34zM8 14A6 6 0 1 0 8 2a6 6 0 0 0 0 12z"></path>
            </svg>
        </div>
    </div>
</div>

<script>
    import axios from 'axios';

    var inputValue = "";
    var baseUrl = 'https://api.labs.cognitive.microsoft.com/academic/v1.0';
    $: dir = '/interpret?query=' + inputValue  + '&complete=0&count=10&model=latest';


    function handleInput(event) {
        inputValue  = event.target.value;   

        if (event.key == "Enter" && inputValue ) {            
            const instance = axios.create({
                baseURL: baseUrl,
                timeout: 1000,
                headers: {
                    'Ocp-Apim-Subscription-Key': '554c1cf34356401ab6fc1dd31982f3ce',
                    'Content-Type': 'application/json'
                },
            });

            instance.get(dir)
                .then(function (response) {
                    console.log(response)
                })
        }
    }

    function callback(error, response, body) {
        if (!error && response.statusCode == 200) {
            var info = JSON.parse(body);
            console.log(info.stargazers_count + " Stars");
            console.log(info.forks_count + " Forks");
        }
    }
</script>