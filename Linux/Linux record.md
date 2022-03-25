###### tags: `Linux`
# Linux record

## clean yum package cache

有可能遇到設定repo之後要換到別的repo的情況, 此時有可能版本卡住, 這時候清掉全部cache應可以解決問題

```
yum clean packages
yum clean headers
yum clean metadata
yum clean all
```

## useful linux command

1. tunnel with ssh (local port 3337 -> remote host's 127.0.0.1:6379)  

`ssh -L 3337:127.0.0.1:6379 root@emkc.org -N`

2. do not add command to history (note the leading space)

` ls -l`
 
3. redo last command bau as root

`sudo !!`

4. quickly create folders

`mkdir -p folder/{sub1,sub2}/{sub1,sub2,sub3}`
`mkdir -p folder/{1...100}/{1...100}`

5. intercept stdout and log to file(截獲stdout輸出, 並複製一份到檔案或目標)

`cat file | tee -a log | cat > /dev/null`

6. exit terminal but leave all processes running

`disown -a && exit

7. fix a long command that you messed up(透過編輯器修改前一次弄錯的指令, 退出後會執行指令)

`fc`