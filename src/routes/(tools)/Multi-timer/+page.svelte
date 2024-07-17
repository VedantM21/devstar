<script lang="ts">
	import { onMount } from 'svelte';

	// ************************************* TIMER CODE **********************************************
	let db;
	let timers = [];
	let alarm;

	let hours = 0;
	let minutes = 0;
	let seconds = 0;
	let remString = "";
	let intervalMap = new Map();
	let showModal = false;
	let modalMessage = "";
	onMount(() => {
		const request = indexedDB.open("TimerDatabase", 1);

		request.onupgradeneeded = (event) => {
			db = request.result;
			const objectStore = db.createObjectStore("times", { keyPath: "id", autoIncrement: true });
			console.log("Object store created");
		}
		request.onsuccess = () => {
			db = request.result;
			console.log("Db opened");
			getAllTimers();
		}
	})

	function getAllTimers(){
		const transaction = db.transaction(["times"], "readonly");
    	const objectStore = transaction.objectStore("times");
    	const getRequest = objectStore.getAll();

		getRequest.onsuccess = (event) => {
			timers = event.target.result;
			console.log(timers);
		}
	}

	function saveTimer(){
		if(hours > 24 || minutes>59 || seconds>59 || remString == ""){
			alert("Please input correct time\nHours not more than 24.\nMinutes not more than 59.\nSeconds not more than 59.\nReminder should not be empty.")
			return;
		}

		const transaction = db.transaction(["times"], "readwrite");
    	const objectStore = transaction.objectStore("times");
		let req;
		req = objectStore.add({ Hours: hours, Minutes: minutes, Seconds: seconds, RemString: remString});
		req.onsuccess = function(event){
			console.log("Timer saved successfully");
		}

		getAllTimers();
	}

	async function startTimer(hrs, min, sec, id) {
		let totalSeconds = hrs * 3600 + min * 60 + sec;
		if(totalSeconds === 0){
			return;
		}

		let timeInterval = setInterval(() => {
			let hours = Math.floor(totalSeconds / 3600);
        	let minutes = Math.floor((totalSeconds % 3600) / 60);
        	let seconds = totalSeconds % 60;
			let timerIndex = timers.findIndex(timer => timer.id === id);

			timers[timerIndex].Hours = hours;
            timers[timerIndex].Minutes = minutes;
            timers[timerIndex].Seconds = seconds;

			timers = timers;

			if(totalSeconds === 0){
				clearInterval(timeInterval);
				showModal = true;
				modalMessage = "Reminder! " + timers[timerIndex].RemString;
				alarm.play();
				resetTimer(id);
				return;
			}


			--totalSeconds;
		}, 1000);

		intervalMap.set(id, timeInterval);
	}

	async function stopTimer(id){
		const intervalID = intervalMap.get(id);
		if (intervalID) {
			clearInterval(intervalID);
			intervalMap.delete(id);
			console.log("Timer stopped: " + id);
		}
	}

	async function start(hrs, min, sec, id){
		const intervalID = intervalMap.get(id);
		if(intervalID){
			return;
		}
		startTimer(hrs, min, sec, id);
	}

	async function resetTimer(id){
		const transaction = db.transaction(["times"], "readonly");
    	const objectStore = transaction.objectStore("times");
		const request = objectStore.get(id);

		request.onsuccess = (event) => {
			const timer = event.target.result;
			let timerIndex = timers.findIndex(t => t.id === id);

			if (timerIndex !== -1) {
				timers[timerIndex].Hours = timer.Hours;
				timers[timerIndex].Minutes = timer.Minutes;
				timers[timerIndex].Seconds = timer.Seconds;

				timers = [...timers];

				stopTimer(id);
			}
		}
	}
	
	async function deleteTimer(id){
		stopTimer(id);
		const transaction = db.transaction(["times"], "readwrite");
    	const objectStore = transaction.objectStore("times");
		const request = objectStore.delete(id);

		request.onsuccess = () => {
			timers = timers.filter(timer => timer.id !== id);
			console.log("Timer deleted: " + id);
		}

	}

	async function closeModal(){
		showModal = false;
		modalMessage = "";
		alarm.pause();
		alarm.currentTime = 0;
	}


	let timer = true;
	let stopWatch = false;

	function selectTimer(){
		console.log("Timer button");
		timer = true;
		stopWatch = false;
	}
	function selectStopWatch(){
		console.log("StopWatch button");
		timer = false;
		stopWatch = true;
	}


	// ************************************** StopWatch Code ****************************************************
	let stopwatchInterval;
	let elapsedTime = 0;
	let stopwatchRunning = false;
	let lapsContainer;
	let lapn=0;

	onMount(() => {
		if (typeof window !== 'undefined') {
			lapsContainer = document.getElementById('laps');
		}
	});
	// lapsContainer = document.getElementById('laps');

	function updateTime() {
		elapsedTime += 10;
		const milliseconds = elapsedTime % 1000;
		const hours = Math.floor(elapsedTime / 3600000);
		const minutes = Math.floor((elapsedTime / 60000) % 60);
		const seconds = Math.floor((elapsedTime / 1000) % 60);
		
		const display = document.getElementById('display');
		display.textContent = 
		String(hours).padStart(2, '0') + ':' + 
		String(minutes).padStart(2, '0') + ':' + 
		String(seconds).padStart(2, '0') + '.' +
		String(milliseconds).padStart(3, '0');
	}

	function toggleStopwatch() {
		const button = document.getElementById('startPauseButton');
		
		if (!stopwatchRunning) {
			stopwatchRunning = true;
			stopwatchInterval = setInterval(updateTime, 10);
			button.textContent = 'Pause';
		} else {
			stopwatchRunning = false;
			clearInterval(stopwatchInterval);
			button.textContent = 'Start';
		}
	}

	function resetStopwatch() {
		stopwatchRunning = false;
		clearInterval(stopwatchInterval);
		elapsedTime = 0;
		lapn=0;
		const display = document.getElementById('display');
		display.textContent = '00:00:00.000';

		const button = document.getElementById('startPauseButton');
		button.textContent = 'Start';

		
		document.getElementById('laps').innerHTML = '';
	}

	function recordLap() {
		lapn+=1;
		const lapTime = document.getElementById('display').textContent;
		const lapElement = document.createElement('div');
		lapElement.textContent = `Lap ${lapn}: ${lapTime}`;
		document.getElementById('laps').appendChild(lapElement);
	}

	
</script>

<div class="card gap-16 items-center mx-auto max-w-screen-xl overflow-hidden rounded-lg p-5">
	<!-- Add tool here -->
	<div class="flex space-x-4 justify-center my-5 flex-wrap">
		<button class="SelectButton bg-gray-800 text-white p-3" on:click={selectTimer}> Timer </button>
		<button class="SelectButton bg-gray-800 text-white p-3" on:click={selectStopWatch}> Stop Watch </button>
	</div>

	{#if showModal}
	<div class="flex justify-center modal">
		<div class="modal-content">
			<p>{modalMessage}</p>
			<button on:click={closeModal}>Close</button>
		</div>
	</div>
	{/if}
	
	<div class="bg-gray-800 p-5 w-full md:w-auto">
		{#if timer}
			<p class="flex text-white justify-center font-bold text-lg"> Timer </p>
			<div class="flex text-black justify-center space-x-2 mt-3 flex-wrap">
				<input bind:value={hours} type="number" min="0" max="24" class="bg-gray-300 border-black border-2 w-20 md:w-32 rounded-lg text-center">
				<p class="text-white text-4xl"> : </p>
				<input bind:value={minutes} type="number" min="0" max="59" class="bg-gray-300 border-black border-2 w-20 md:w-32 rounded-lg text-center">
				<p class="text-white text-4xl"> : </p>
				<input bind:value={seconds} type="number" min="0" max="59" class="bg-gray-300 border-black border-2 w-20 md:w-32 rounded-lg text-center">
			</div>
			<div class="flex justify-center">
				<p class="text-white font-bold text-3xl mt-3 mx-5">Reminder: </p>
				<input bind:value={remString} class="bg-gray-300 border-black border-2 w-40 md:w-64 rounded-lg text-center p-2 my-2" minlength="0" maxlength="20" type="text">
			</div>

			<div class="flex justify-center mt-5">
				<button on:click={saveTimer} class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"> Save </button>
			</div>

			<audio bind:this={alarm} src="src\routes\(tools)\Multi-timer\alarm.mp3"></audio>

			<p class="flex justify-center text-4xl text-white pt-10 py-5"> Saved Timers </p>

			{#each timers as t}
			<div class="bg-gray-700 flex flex-wrap justify-center border-2 rounded-lg border-black mx-10 my-3 p-2">
				<p class="text-center text-2xl text-white p-2 md:mt-2">{t.RemString}</p>
				<div class="flex items-center space-x-2 md:mt-1">
					<p class="bg-gray-300 border-black border-2 w-20 md:w-32 rounded-lg text-center p-2">{t.Hours}</p>
					<p class="text-white text-4xl">:</p>
					<p class="bg-gray-300 border-black border-2 w-20 md:w-32 rounded-lg text-center p-2">{t.Minutes}</p>
					<p class="text-white text-4xl">:</p>
					<p class="bg-gray-300 border-black border-2 w-20 md:w-32 rounded-lg text-center p-2">{t.Seconds}</p>
				</div>
				<div class="flex flex-wrap justify-center mt-1 md:ml-5">
					<button on:click={() => start(t.Hours, t.Minutes, t.Seconds, t.id)} class="mx-2 my-2 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">Start</button>
					<button on:click={() => stopTimer(t.id)} class="mx-2 my-2 bg-yellow-500 hover:bg-yellow-700 text-white font-bold py-2 px-4 rounded">Stop</button>
					<button on:click={() => resetTimer(t.id)} class="mx-2 my-2 bg-green-500 hover:bg-green-700 text-white font-bold py-2 px-4 rounded">Reset</button>
					<button on:click={() => deleteTimer(t.id)} class="mx-2 my-2 bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded">Delete</button>
				</div>
			</div>									
			
			{/each}

			
		{:else}
			<div class="bg-[#323A49] w-full md:w-2/3 text-center p-4 rounded-lg mx-auto">
				<h1 id="display"  class="text-white text-4xl my-6">00:00:00.000</h1>
				<div class="flex space-x-4 justify-center my-10 text-lg flex-wrap">
					<button id="startPauseButton" on:click={toggleStopwatch} class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">Start</button>
					<button on:click={resetStopwatch} class="bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded">Reset</button>
					<button on:click={recordLap} class="bg-yellow-500 hover:bg-yellow-700 text-white font-bold py-2 px-4 rounded">Lap</button>
				</div>
				<div id="laps" class="text-lg md:text-xl my-10 text-white"></div>
			</div>	
		{/if}
	</div>


</div>

<style>
	/* .SelectButton {
		display: flex;
		justify-content: center;
		gap: 10px;
		margin-bottom: 20px;
	}

	button {
		padding: 10px 20px;
		background-color: #007bff;
		color: white;
		border: none;
		border-radius: 5px;
		cursor: pointer;
		transition: background-color 0.3s;
	} */

	button:hover {
		background-color: #0056b3;
	}

	/* Modal styles */
	.modal {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background-color: rgba(0, 0, 0, 0.5);
		display: flex;
		justify-content: center;
		align-items: center;
		z-index: 1000; 
	}

	.modal-content {
		background-color: white;
		padding: 20px;
		border-radius: 8px;
		text-align: center;
	}

	.modal-content button {
		margin-top: 10px;
		padding: 8px 16px;
		background-color: #007bff;
		color: white;
		border: none;
		border-radius: 5px;
		cursor: pointer;
		transition: background-color 0.3s;
	}

	.modal-content button:hover {
		background-color: #0056b3;
	}
</style>