let elapsed = Date.now() - startTime;

let hours = Math.floor(elapsed / 3600000);
let minutes = Math.floor((elapsed % 3600000) / 60000);
let seconds = Math.floor((elapsed % 60000) / 1000);

let duration = `${hours}h ${minutes}m ${seconds}s`;
