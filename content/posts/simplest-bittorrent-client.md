---
title: Simplest bittorrent client
date: 2018-01-21T01:50:24+09:00
tags: ["bittorrent", "webtorrent", "javascript"]
---

WebTorrentというライブラリを使えば、javascriptで簡単にbittorrentクライアントを構築することができる！！！！！！！！！！

{{< highlight html >}}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>WebTorrent Test</title>
    <style>
      input {
        width: 500px;
      }
    </style>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css" integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm" crossorigin="anonymous">
  </head>
  <body>
    <br>
    <div class="container-fluid" id="input-form">
      <label for="url">URL</label>
      <div class="input-group">
        <input type="text" id="url" class="form-control" placeholder="url" aria-label="url">
      </div>
      
      <br>
      
      <label for="path">PATH</label>
      <div class="input-group">
        <input type="text" id="path" class="form-control" placeholder="path" aria-label="path">
      </div>
      
      <br>
      
      <button id="go" class="btn btn-secondary" type="button">GO!</button>
    </div>

    <div class="container-fluid" id="progress">
      <div class="progress">
        <div id="progress-bar" class="progress-bar" role="progressbar" aria-valuenow="0" aria-valuemin="0" aria-valuemax="100"></div>
      </div>
      <br>
    </div>

    <script src="index.js"></script>
  </body>
</html>
{{< / highlight >}}

{{< highlight javascript >}}
let WebTorrent = require('webtorrent')
    $ = require('jquery')

$("#progress").css("display", "none")

$("#path").val("/tmp/webtorrent/")

$("#go").click(() => {
  $("#input-form").remove()
  $("#progress").css("display", "block")
  
  let client = new WebTorrent()

  let torrentId = $("#url").val()

  let path = $("#path").val()

  client.add(torrentId, {path}, function (torrent) {
    let update = () => {
      let current_progress = Math.floor(torrent.progress * 100)
      console.log(current_progress)
      $("#progress-bar").attr("aria-valuenow", current_progress)
      $("#progress-bar").text(current_progress + "%")
      $("#progress-bar").css("width", current_progress + "%")
    }
    
    setInterval(update, 1000 / 24)
    
    let file = torrent.files.find(function (file) {
      console.log(file)
      return file.name.endsWith('.mp4')
    })

    file.appendTo('#progress')
  })
  
  client.on("error", (e) => {
    $('#progress').append('<div class="alert alert-danger">' + e + '</div>')
  })
})
{{< / highlight >}}
