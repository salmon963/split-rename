# 問題描述
原始的語料資料為逐句錄製的語音檔案，每個檔案按照「段落編號+句子編號」進行命名（如：段落一第一句.wav）。然而，所提供的文字語料檔案並未對語句進行相同的分割，而是將整個語料內容合併為一個完整的文本文件，缺乏對應的結構分段，因此，我們需要將文字檔案進行分割，根據語音檔案的段落編號與句子編號（如「段落X第Y句」）對應切割文字內容。分割後的文字內容與對應的錄音檔按照統一的命名規則命名，以便用於後續模型訓練的需求。
分割前：
![image](https://hackmd.io/_uploads/H1BtGk4Ekg.png)
分割後：
![image](https://hackmd.io/_uploads/S1kTfJE4Jx.png)
![image](https://hackmd.io/_uploads/H1OyQJN4Jx.png)
錄音檔重新命名前：
![image](https://hackmd.io/_uploads/B1Hi7JEEye.png)
錄音檔重新命名後：
![image](https://hackmd.io/_uploads/SkGB7JVN1e.png)
## 程式目的
此程式會ㄣ根據輸入的文字檔（TXT 文檔），按照需求分割內容並重新命名輸出檔案，可以根據需求更改此程式中的分割、命名規則來達成目標。此次目標是根據文字中的「段落 X 第 Y 句」結構，提取對應的 X（段落編號）與 Y（句子編號），並生成新的檔案名稱與內容
檔案命名規則：
* 檔案命名格式為：xs1xy.txt
    * 其中：
        * X 是段落編號，來自中文數字，轉換為阿拉伯數字
        * Y 是句子編號，來自中文數字，轉換為阿拉伯數字
        * x、y輸出為：01、02...10、11...(目的為保持數字為5碼)
* 示例：
    * 「段落一，第四句」 將生成檔案名稱 xs10104.txt
    * 「段落十一，第十一句」 將生成檔案名稱 xs11111.txt
* 程式運作範例：
    * 輸入文檔為：
    ```
    段落一，第一句：這是第一句話。
    段落一，第十二句：這是第十二句話。
    段落二，第一句：這是新段落的第一句話
    段落十一，第一句：這是段落十一的第一句話。
    ```
    * 生成的檔案會是：
        * xs10101.txt，內容：段落一，第一句：這是第一句話。
        * xs10112.txt，內容：段落一，第十二句：這是第十二句話。
        * xs10201.txt，內容：段落二，第一句：這是新段落的第一句話
        * xs11101.txt，段落十一，第一句：這是段落十一的第一句話。
## 程式碼解釋
### 完整程式碼
```c=
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

// 將中文數字轉換為阿拉伯數字
int chinese_to_digit(const char *chinese_num) {
    int result = 0;
    const char *ten_ptr = strstr(chinese_num, "十");
    if (ten_ptr) {
        // 處理十位數部分
        if (ten_ptr == chinese_num) {  
            result += 10;
        } 
        // 處理個位數部分
        if (*(ten_ptr + strlen("十")) != '\0') {
            result += chinese_to_digit(ten_ptr + strlen("十"));
        }
    } else {
        if (strcmp(chinese_num, "一") == 0) return 1;
        if (strcmp(chinese_num, "二") == 0) return 2;
        if (strcmp(chinese_num, "三") == 0) return 3;
        if (strcmp(chinese_num, "四") == 0) return 4;
        if (strcmp(chinese_num, "五") == 0) return 5;
        if (strcmp(chinese_num, "六") == 0) return 6;
        if (strcmp(chinese_num, "七") == 0) return 7;
        if (strcmp(chinese_num, "八") == 0) return 8;
        if (strcmp(chinese_num, "九") == 0) return 9;
        if (strcmp(chinese_num, "十") == 0) return 10;  
    }
    return result;
}

// 將文本按段落和句子分割並保存到不同文件的函數
void splitTextByParagraph(const char *txtPath) {
    FILE *inputFile = fopen(txtPath, "r");
    if (inputFile == NULL) {
        perror("無法打開文件");
        return;
    }

    char line[1024];
    int paragraph_num = 0, sentence_num = 0;
    FILE *outputFile = NULL;
    char output_filename[50];

    while (fgets(line, sizeof(line), inputFile) != NULL) {
        // 檢查是否為新段落的開始
        const char *para_ptr = strstr(line, "段落");
        const char *sent_ptr = strstr(line, "第");
        const char *sent_end_ptr = strstr(line, "句");

        // 若找到 "段落"、"第" 和 "句" 關鍵字，表示是新段落和句子的開始
        if (para_ptr && sent_ptr && sent_end_ptr && sent_ptr < sent_end_ptr) {
            // 提取段落編號
            char paragraph_chinese[10] = {0};
            const char *para_end_ptr = strstr(para_ptr, "，"); // 找到 "段落" 後的 "，" 作為結束標記
            if (para_end_ptr) {
                strncpy(paragraph_chinese, para_ptr + strlen("段落"), para_end_ptr - (para_ptr + strlen("段落")));
                paragraph_chinese[para_end_ptr - (para_ptr + strlen("段落"))] = '\0';
                paragraph_num = chinese_to_digit(paragraph_chinese);
            } else {
                continue;
            }

            // 提取句子編號
            char sentence_chinese[10] = {0};
            strncpy(sentence_chinese, sent_ptr + strlen("第"), sent_end_ptr - (sent_ptr + strlen("第")));
            sentence_chinese[sent_end_ptr - (sent_ptr + strlen("第"))] = '\0';
            sentence_num = chinese_to_digit(sentence_chinese);

            // 關閉上一個段落的輸出文件
            if (outputFile != NULL) {
                fclose(outputFile);
            }

            // 根據段落和句子編號生成新的檔案名稱
            snprintf(output_filename, sizeof(output_filename), "xs10%d0%d.txt", paragraph_num, sentence_num);
            outputFile = fopen(output_filename, "w");

            if (outputFile == NULL) {
                perror("無法創建輸出文件");
                fclose(inputFile);
                return;
            }

            printf("創建文件: %s\n", output_filename); // 顯示創建的文件名
        }

        // 將當前行寫入當前的輸出文件
        if (outputFile != NULL) {
            fputs(line, outputFile);
        }
    }

    // 關閉最後一個段落的輸出文件
    if (outputFile != NULL) {
        fclose(outputFile);
    }

    // 關閉輸入文件
    fclose(inputFile);
}

// 主函數
int main() {
    char txtPath[256];

    printf("請輸入文字檔案的路徑：");
    scanf("%255s", txtPath);

    splitTextByParagraph(txtPath);

    printf("文字檔已按照段落和句子輸出至各自的檔案\n");
    return 0;
}

```
### 1 標頭檔
```c=
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
```
### 2 中文數字轉換函數
```c=
int chinese_to_digit(const char *chinese_num) {
    int result = 0;
    const char *ten_ptr = strstr(chinese_num, "十");
    if (ten_ptr) {
        if (ten_ptr == chinese_num) {  
            result += 10;
        } 
        if (*(ten_ptr + strlen("十")) != '\0') {
            result += chinese_to_digit(ten_ptr + strlen("十"));
        }
    } else {
        if (strcmp(chinese_num, "一") == 0) return 1;
        if (strcmp(chinese_num, "二") == 0) return 2;
        if (strcmp(chinese_num, "三") == 0) return 3;
        if (strcmp(chinese_num, "四") == 0) return 4;
        if (strcmp(chinese_num, "五") == 0) return 5;
        if (strcmp(chinese_num, "六") == 0) return 6;
        if (strcmp(chinese_num, "七") == 0) return 7;
        if (strcmp(chinese_num, "八") == 0) return 8;
        if (strcmp(chinese_num, "九") == 0) return 9;
        if (strcmp(chinese_num, "十") == 0) return 10;  
    }
    return result;
}
```
* 功能： 將中文數字轉換為對應的阿拉伯數字
* 主要邏輯：
    * 1.使用 strstr 搜尋字串中是否有「十」
    * 2.若有「十」在字串中
        * 當「十」位於開頭（表示十位數），則累加 10
        * 當「十」後方仍有字元，處理其餘部分
    * 3.如果找不到「十」，用 strcmp 判斷字串是否與個別數字一致。 
* 語法：
    * strcmp：比較兩字串是否相等。
    * strstr：尋找字串中指定子字串的位置。
### 3 分段與分句函數
#### 3-1 函數開頭
```c=
void splitTextByParagraph(const char *txtPath) {
    FILE *inputFile = fopen(txtPath, "r");
    if (inputFile == NULL) {
        perror("無法打開文件");
        return;
    }
```
* 功能：打開輸入檔案
#### 3-2 讀取檔案內容並判斷段落與句子
```c=
    char line[1024];
    int paragraph_num = 0, sentence_num = 0;
    FILE *outputFile = NULL;
    char output_filename[50];

    while (fgets(line, sizeof(line), inputFile) != NULL) {
        const char *para_ptr = strstr(line, "段落");
        const char *sent_ptr = strstr(line, "第");
        const char *sent_end_ptr = strstr(line, "句");

        if (para_ptr && sent_ptr && sent_end_ptr && sent_ptr < sent_end_ptr) {
```
* 功能：每次讀取檔案的一行，尋找特定的關鍵字（"段落"、"第"、"句"）以確定是否進入新段落
* 細項討論
    ```c
    while (fgets(line, sizeof(line), inputFile) != NULL) {
    ``` 
    * 使用 fgets 從檔案中讀取一行文字，存入 line。
    * 如果檔案已讀取完畢，fgets 會返回 NULL，跳出 while 迴圈
    ```c
        const char *para_ptr = strstr(line, "段落");
        const char *sent_ptr = strstr(line, "第");
        const char *sent_end_ptr = strstr(line, "句");
    ```
    * 使用 strstr 函數在 line 中查找特定的關鍵字：
        * "段落"：標記段落的起點。
        * "第"：句子的編號起點。
        * "句"：句子的結尾。
    ```c
    if (para_ptr && sent_ptr && sent_end_ptr && sent_ptr < sent_end_ptr) {
    ``` 
    * 判斷是否是新段落和句子
        * para_ptr、sent_ptr、sent_end_ptr 均不為 NULL：表示當前行同時包含 "段落"、"第" 和 "句" 關鍵字。
        * sent_ptr < sent_end_ptr：確保 "第" 出現的位置在 "句" 之前，避免誤判。
#### 3-3 提取段落與句子編號 
```c=
            char paragraph_chinese[10] = {0};
            const char *para_end_ptr = strstr(para_ptr, "，");
            if (para_end_ptr) {
                strncpy(paragraph_chinese, para_ptr + strlen("段落"), para_end_ptr - (para_ptr + strlen("段落")));
                paragraph_num = chinese_to_digit(paragraph_chinese);
            } else {
                continue;
            }

            char sentence_chinese[10] = {0};
            strncpy(sentence_chinese, sent_ptr + strlen("第"), sent_end_ptr - (sent_ptr + strlen("第")));
            sentence_num = chinese_to_digit(sentence_chinese);

```
* 功能： 根據中文數字解析段落和句子編號。
* 細項討論：
    ```c
        char paragraph_chinese[10] = {0};
        const char *para_end_ptr = strstr(para_ptr, "，");
        if (para_end_ptr) {
            strncpy(paragraph_chinese, para_ptr + strlen("段落"), para_end_ptr - (para_ptr + strlen("段落")));
            paragraph_num = chinese_to_digit(paragraph_chinese);
        } else {
            continue;
        }
    ```
    * 提取段落中的中文數字（如 "段落三" 中的 "三"），將其轉換為對應的阿拉伯數字。
        ```c 
        const char *para_end_ptr = strstr(para_ptr, "，"); 
        ```
        * 使用 strstr 尋找 para_ptr 指向的字串中，第一個出現的，的位置
        ```c 
        strncpy(paragraph_chinese, para_ptr + strlen("段落"), para_end_ptr - (para_ptr + strlen("段落")));
        ```        
        * 使用 strncpy 從 para_ptr 的位置提取中文數字部分
        * 起點為 "段落" 之後的位置（para_ptr + strlen("段落")）。
        * 長度為 para_end_ptr - (para_ptr + strlen("段落"))，即從 "段落" 的結束到 ， 的起始之間的字元數。
        ```c
        paragraph_num = chinese_to_digit(paragraph_chinese);
        ```
        * 呼叫 chinese_to_digit 函數，將提取到的中文數字轉換為對應的整數（如 "三" 轉換為 3）。
    ```c
        char sentence_chinese[10] = {0};
        strncpy(sentence_chinese, sent_ptr + strlen("第"), sent_end_ptr - (sent_ptr + strlen("第")));
        sentence_num = chinese_to_digit(sentence_chinese);
    ```
    * 提取句子中的中文數字（如 "第一句" 中的 "一"），並轉換為對應的阿拉伯數字。邏輯、語法與提取段落相同
#### 3-4 創建輸出檔案
```c=
            if (outputFile != NULL) {
                fclose(outputFile);
            }

            snprintf(output_filename, sizeof(output_filename), "xs10%d0%d.txt", paragraph_num, sentence_num);
            outputFile = fopen(output_filename, "w");

            if (outputFile == NULL) {
                perror("無法創建輸出文件");
                fclose(inputFile);
                return;
            }

            printf("創建文件: %s\n", output_filename);
        }
```
* 功能：建立新檔案名稱，印出檔案名稱，顯示檔案建立成功
### 3-5
```c=
        if (outputFile != NULL) {
            fputs(line, outputFile);
        }
    }

    if (outputFile != NULL) {
        fclose(outputFile);
    }
    fclose(inputFile);
}
```
* 功能： 將目前讀取的行寫入對應的輸出檔案，並在完成處理後關閉所有檔案。
### 4 主程式
```c=
int main() {
    char txtPath[256];

    printf("請輸入文字檔案的路徑：");
    scanf("%255s", txtPath);

    splitTextByParagraph(txtPath);

    printf("文字檔已按照段落和句子輸出至各自的檔案\n");
    return 0;
}
```
* 要求用戶輸入檔案路徑，並呼叫分段函數處理內容。
## 執行方法&測試案例
編譯：
```
gcc -o split split.c
```
執行：
```
./split 
```
會跳出請輸入文字檔的路徑 輸入即可完成
![image](https://hackmd.io/_uploads/BktIpdWQke.png)

# Rename
## 程式目的
此程式的功能是將資料夾內的音檔名稱進行重新命名，將符合特定規則的 .wav 檔案名(段落X第Y句.wav)提取對應的 X（段落編號）與 Y（句子編號），轉換檔案名稱
檔案命名規則：
* 檔案命名格式為：xs1xy.txt
    * 其中：
        * X 是段落編號，來自中文數字，轉換為阿拉伯數字
        * Y 是句子編號，來自中文數字，轉換為阿拉伯數字
        * x、y輸出為：01、02...10、11...(目的為保持數字為5碼)
* 示例：
    * 「段落一第四句.wav」 將生成檔案名稱 xs10104.wav
    * 「段落十一第十一句.wav」 將生成檔案名稱 xs11111.wav

## 程式碼解釋
### 完整程式碼
```c=
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>

// 將中文數字轉換為阿拉伯數字
int chinese_to_digit(const char *chinese_num) {
    int result = 0;
    const char *ten_ptr = strstr(chinese_num, "十"); // 尋找「十」在字串中的位置
    if (ten_ptr) {
        // 處理十位數部分
        result += 10;

        // 處理個位數部分
        if (*(ten_ptr + strlen("十")) != '\0') {
            result += chinese_to_digit(ten_ptr + strlen("十"));
        }
    } else {
        // 處理純個位數
        if (strcmp(chinese_num, "一") == 0) return 1;
        if (strcmp(chinese_num, "二") == 0) return 2;
        if (strcmp(chinese_num, "三") == 0) return 3;
        if (strcmp(chinese_num, "四") == 0) return 4;
        if (strcmp(chinese_num, "五") == 0) return 5;
        if (strcmp(chinese_num, "六") == 0) return 6;
        if (strcmp(chinese_num, "七") == 0) return 7;
        if (strcmp(chinese_num, "八") == 0) return 8;
        if (strcmp(chinese_num, "九") == 0) return 9;
        if (strcmp(chinese_num, "十") == 0) return 10;
    }
    return result;
}

// 更改檔案名稱並記錄錯誤訊息
void rename_file(const char* input_folder, const char* filename) {
    char paragraph[10] = {0}, sentence[10] = {0};
    int paragraph_num = 0, sentence_num = 0;

    const char *para_ptr = strstr(filename, "段落");
    const char *sent_ptr = strstr(filename, "第");
    const char *sent_end_ptr = strstr(filename, "句");

    if (para_ptr && sent_ptr && sent_end_ptr && sent_ptr < sent_end_ptr) {
        strncpy(paragraph, para_ptr + strlen("段落"), sent_ptr - (para_ptr + strlen("段落")));
        paragraph[sent_ptr - (para_ptr + strlen("段落"))] = '\0';
        paragraph_num = chinese_to_digit(paragraph);

        strncpy(sentence, sent_ptr + strlen("第"), sent_end_ptr - (sent_ptr + strlen("第")));
        sentence[sent_end_ptr - (sent_ptr + strlen("第"))] = '\0';
        sentence_num = chinese_to_digit(sentence);
    } else {
        printf("檔案名稱格式錯誤: %s\n", filename);
        return;
    }

    if (paragraph_num == 0 || sentence_num == 0) {
        printf("無效的數字轉換結果於檔案: %s\n", filename);
        return;
    }

    char new_filename[30];
    // 修正段落和句子補零的邏輯
    snprintf(new_filename, sizeof(new_filename), "xs1%02d%02d.wav", paragraph_num, sentence_num);

    char old_filepath[256], new_filepath[256];
    snprintf(old_filepath, sizeof(old_filepath), "%s/%s", input_folder, filename);
    snprintf(new_filepath, sizeof(new_filepath), "%s/%s", input_folder, new_filename);

    if (rename(old_filepath, new_filepath) == 0) {
        printf("檔案已重新命名為: %s\n", new_filename);
    } else {
        perror("無法重新命名檔案");
    }
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "請提供資料夾路徑\n");
        return 1;
    }

    const char *input_folder = argv[1];

    DIR *dp = opendir(input_folder);
    if (dp == NULL) {
        perror("無法打開資料夾");
        return 1;
    }

    struct dirent *entry;
    while ((entry = readdir(dp))) {
        if (strstr(entry->d_name, ".wav")) {
            rename_file(input_folder, entry->d_name);
        }
    }

    closedir(dp);
    return 0;
}
```
### 1.中文數字轉換函數、提取段落與句子編號
這部分內容的邏輯與 split 中語法相似，因此省略詳細說明
### 2.構建新檔案名稱並重命名
```c=
    if (paragraph_num == 0 || sentence_num == 0) {
        printf("無效的數字轉換結果於檔案: %s\n", filename);
        return;
    }

    char new_filename[30];
    snprintf(new_filename, sizeof(new_filename), "xs1%02d%02d.wav", paragraph_num, sentence_num);
```
* 使用 snprintf 構建新檔案名稱，格式為 xs1xxYY.wav（xx 和 YY 分別代表段落和句子的兩位數字）。
### 重新命名
```c=
    char old_filepath[256], new_filepath[256];
    snprintf(old_filepath, sizeof(old_filepath), "%s/%s", input_folder, filename);
    snprintf(new_filepath, sizeof(new_filepath), "%s/%s", input_folder, new_filename);

    if (rename(old_filepath, new_filepath) == 0) {
        printf("檔案已重新命名為: %s\n", new_filename);
    } else {
        perror("無法重新命名檔案");
    }
}
```
* 若重命名成功，打印新檔案名稱；否則打印錯誤訊息
### 主程式
```c=
int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "請提供資料夾路徑\n");
        return 1;
    }

    const char *input_folder = argv[1];

    DIR *dp = opendir(input_folder);
    if (dp == NULL) {
        perror("無法打開資料夾");
        return 1;
    }

    struct dirent *entry;
    while ((entry = readdir(dp))) {
        if (strstr(entry->d_name, ".wav")) {
            rename_file(input_folder, entry->d_name);
        }
    }

    closedir(dp);
    return 0;
}
```
* 功能：使用 opendir 開啟資料夾，並用 readdir 遍歷所有檔案，遇到副檔名為 .wav 的檔案，調用 rename_file 處理
*  細項討論：
    ```
    int main(int argc, char *argv[]) {
        if (argc < 2) {
            fprintf(stderr, "請提供資料夾路徑\n");
            return 1;
        }
    ```
    * 檢查命令行參數是否正確，若未提供資料夾路徑則打印錯誤
    ```
    struct dirent *entry;
    while ((entry = readdir(dp))) {
        if (strstr(entry->d_name, ".wav")) {
            rename_file(input_folder, entry->d_name);
        }
    }
    ```
    功能：
    * 1.讀取資料夾內容：
        * 使用 readdir 逐個讀取資料夾中的檔案或目錄。
        * 每次執行 readdir 都返回一個 struct dirent 結構體指針，表示資料夾中的一個項目（例如檔案或子目錄）
        * 當 readdir 返回 NULL 時，表示已經讀取完所有內容。
    * 2.過濾檔案：
        * entry->d_name 是目前檔案或目錄的名稱。
        * 使用 strstr(entry->d_name, ".wav") 判斷名稱中是否包含.wav，即篩選出 .wav 格式的檔案
    * 3.對於符合條件的 .wav 檔案，調用 rename_file 函數進行處理
## 執行方法&測試案例
編譯：
```
gcc -o rename rename.c
```
執行：
```
./rename <資料夾路徑>
```
![image](https://hackmd.io/_uploads/B11j8jbm1x.png)

