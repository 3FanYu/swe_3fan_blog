# Docker Sync


在現任公司開發時發現，原本設計主repo的前輩是希望大家使用 docker 來建制開發環境，這樣一來除了可以使用docker-compose 快速將開發環境跑起來以外，還可以讓開發環境盡量貼近線上環境，減少因為環境不同而產出的一些額外問題。但是開發過程中發現 docker for mac 的速度真的是讓人感到憂鬱，除了會讓我的電腦風扇起飛以外，在container裡面使用 rails console 真的是慢到很痛苦，平均每次 reload 都需要 5 秒的時間。也因此許多資深前輩們都選擇將開發環境移至本地端，這樣的做法雖然讓開發時的程式執行速度提升了不少，但也違背了當初設計repo的初衷，有可能會延伸出其他問題。

而菜雞如我認為，如果將開發環境移至本地端，將會失去許多處理docker問題，更深入理解docker的機會，這樣其實是蠻浪費的。詢問了一下google大神後得知，mac os 之所以會有此等狀況發生，問題來自於 docker for mac 在同步掛載(volume) 的速度上實在太慢了，這也就是為什麼 rails console reload 速度會這麼慢的原因。為了解決這個問題，我們可以使用 docker-sync 來加快 docker 同步掛載資源的速度。

其原理大致上是多起一個 container (這邊先叫它sync-container)，建立 /host_sync 的路徑並使用 OSXFS 的方式去與 host 的檔案做綁定。在同一個 sync-container 內部，再建立一個 /app_sync 的路徑，使用 Unison 去與 host_sync 建立雙向同步 (2-way-sync)，而我們實際運行程式的 container (app-container) 讀取的 volume 來源就來自於 sync-container 的 /app_sync 路徑。這樣一來，app-container 同步 /app_sync 的行為就不會直接受 /host_sync 與 OSXFS 的影響，而是以稍微延遲一點且非同步的方式執行。另外，值得一提的是，在最後的一步，app-container 讀取 /app_sync 進行同步時，因為都是同在container內發生的行為，(也就是 hyperkit)，所以這一步行為並不會受限於本篇一開始所提到的 mac os 同步掛載的坑，據docker-sync 作者的說法，這個步驟是如同 natvie speed 一樣快的。
![example img](./native_osx.png)


1. 安裝

    ```bash
    gem install docker-sync
    ```

2. 於專案目錄下建立docker-sync.yml
   範例：
    ```yml
    version: "2"
    options:
        verbose: true
    syncs:
    #IMPORTANT: ensure this name is unique and does not match your other application container name
    rails-sync: #tip: add -sync and you keep consistent names as a convention
        src: '../rails'
    sync_strategy: 'rsync'
    ```

3. 修改docker-compose.yml
   範例：
   ```yml
    rails:
      image: rails
      volumes:
      # - ../rails:/app
        - rails-sync:/app:rw
      ports:
        - 3000:3000

    volumes:
      rails-sync:
        external: true
    # ...etc
   ```

4. 啟動服務
   先啟動 docker-sync:
   ```cmd
   docker-sync start
   ```
   再啟動其他containers:
   ```cmd
   docker-compose up
   ```

