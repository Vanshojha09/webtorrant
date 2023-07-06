<script>
    import { onMount } from "svelte";
    import dragDrop from 'drag-drop'
    var client
    onMount(async()=>{
      const wt = await import("webtorrent/dist/webtorrent.min");
      const WebTorrent = wt.default;
      client = new WebTorrent();
  
      if (!WebTorrent.WEBRTC_SUPPORT) {
        alert('This browser is unsupported. Please use a browser with WebRTC support.')
      }
      client.on('error', err => {
        console.error('ERROR: ' + err.message)
      })
  
      dragDrop(el, files => {
        console.log(files);
        client.seed(files, torrent => {
          console.log(torrent);
          console.log('Client is seeding:', torrent.magnetURI);
          uri= torrent.magnetURI;
        })
      })
  
      if ('registerProtocolHandler' in navigator) {
        navigator.registerProtocolHandler('magnet', window.location.origin + '#%s', 'Instant.io')
      }
  
      if ('serviceWorker' in navigator) {
        navigator.serviceWorker.register('/sw.js')
      }
    });
  
    const printProcess = (torrentInfo, progressBarLength = 25) => {
    const percent = torrentInfo.progress * 100 | 0;
    const speed = torrentInfo.downloadSpeed / 1024 / 1024 | 0;
    const downloadedSize = torrentInfo.downloaded / 1024 / 1024 | 0;
    const fullSize = torrentInfo.length / 1024 / 1024 | 0;
  
    const percentPerDot = 100 / progressBarLength | 0;
    const doneCount = percent / percentPerDot | 0;
    const done = '\\033[42m' + ' '.repeat(doneCount) + '\\033[0m';
    const undone = ' '.repeat(progressBarLength - doneCount);
  
    console.log(`\r[${done}${undone}] ${percent}% (${downloadedSize}/${fullSize} MB) speed: ${speed} MB/S`);
  };
  
  
    const upload=(e)=>{
      var file = e.target.files[0];
      client.seed(file, torrent => {
          console.log(torrent);
          console.log('Client is seeding:', torrent.magnetURI)
          uri= torrent.magnetURI;
          client.add(uri, onTorrent)
        })
    }
    var el;
    var progress = 0;
    const download=()=>{
      client.add(uri, onTorrent)
    }
  
    async function onTorrent (torrent) {
      console.log('Got torrent metadata!')
      console.log(torrent);
      console.log(
        'Torrent info hash: ' + torrent.infoHash + ' ' +
        '<a href="' + torrent.magnetURI + '" target="_blank">[Magnet URI]</a> ' +
        '<a href="' + URL.createObjectURL(torrent.torrentFileBlob) + '" target="_blank" download="' + torrent.name + '.torrent">[Download .torrent]</a>'
      )
  
      // Print out progress every 5 seconds
      const interval = setInterval(() => {
        progress = (torrent.progress * 100).toFixed(1);
        const speed = torrent.downloadSpeed / 1024 / 1024 | 0;
        const downloadedSize = torrent.downloaded / 1024 / 1024 | 0;
        const fullSize = torrent.length / 1024 / 1024 | 0;
  
        console.log('Progress: ' + progress + '%')
        console.log(torrent);
        console.log(speed, downloadedSize);
      }, 500)
   
      torrent.on('done', () => {
        progress = 100;
        console.log('Progress: '+progress+'%')
        clearInterval(interval)
      })
  
      // Render all files into to the page
      for (const file of torrent.files) {
        try {
          // console.log(file);
          const blob = await file.blob()
          // console.log(blob);
          const url = window.URL.createObjectURL(blob);
          var fileData={
            name: file.name,
            size: file.size,
            type: file.type,
            downloadURL: url
          }
          links= [...links, fileData];
          // console.log(links);
          // console.log('(Blob URLs only work if the file is loaded from a server. "http//localhost" works. "file://" does not.)')
          // console.log('File done.')
          // console.log('<a href="' + URL.createObjectURL(blob) + '">Download full file: ' + file.name + '</a>')
        } catch (err) {
          if (err) console.log(err.message)
        }
      }
    }
  
      var dataEL;
      var links=[];
      var magnet="";
      var uri="";
  </script>
  <div>
    <input type="text" bind:value={magnet} class="border-2 m-4 w-4/5">
    <button on:click|preventDefault={download} class="border-2 bmg bg-gray-200 px-4 py-1 rounded-lg">DOWNLOAD</button>
  </div>
  
  <div class="m-4 p-2">
    <div class="w-full bg-gray-50 rounded-full">
      <div class="bg-blue-600 text-xs font-medium text-gray-600 text-center p-0.5 leading-none rounded-r-full rounded-l-full" style="width: {progress}%"> {progress}%</div>
    </div>
  </div>
  
  <div bind:this={el} class=" h-40 bg-gray-100 m-4 flex justify-center items-center">
    <input type="file" on:change={upload} class=""> 
    <p class="">
      Drag and drop file to start seeding...
    </p>
  </div>     
  
  <p class="mx-4">
    {uri}
  </p>
  <button class="{uri?"":"hidden"} bg-gray-200 px-4 py-1 rounded-lg mx-4" on:click={()=>{navigator.clipboard.writeText(uri)}}>Copy url</button>
  
  <div class="m-4">
    <p>Download files-</p>
    <div bind:this={dataEL}>
      {#each links as link, i}
      <a download="{link.name}" target="_blank" rel="noreferrer" href={link.downloadURL}> {i+1}. {link.name} </a>
      <br/>
      {/each}
    </div>
  </div> 
