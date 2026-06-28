#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#include <windows.h>
#include <shlwapi.h>

#pragma comment(lib, "shlwapi.lib")

#define MAX_EXCLUDES 20

// 文字コードの定義
typedef enum {
    ENC_AUTO,  // 自動判定（デフォルト）
    ENC_UTF8,  // UTF-8固定
    ENC_CP932  // CP932固定
} FileEncoding;

typedef struct {
    char searchStr[256];
    char baseDir[MAX_PATH];
    char filePattern[MAX_PATH];
    char excludeDirs[MAX_EXCLUDES][MAX_PATH];
    int excludeCount;
    bool recursive;
    FileEncoding encoding; // 【追加】文字コード設定
} SearchConfig;

void ParseArguments(int argc, char *argv[], SearchConfig *config);
void SearchDirectory(const char *dirPath, const SearchConfig *config);
void SearchInFile(const char *filePath, const char *searchStr, int baseDirLen, FileEncoding configEnc);
bool IsValidUTF8(const unsigned char *str, size_t len);
void ConvertUTF8toCP932(const char *utf8Src, char *cp932Dest, size_t destSize);
bool isExcluded(const char *dirName, const SearchConfig *config);

int main(int argc, char *argv[]) {
    SearchConfig config;
    memset(&config, 0, sizeof(config));
    
    strcpy(config.baseDir, ".");       
    strcpy(config.filePattern, "*.*"); 
    config.encoding = ENC_AUTO; // デフォルトは自動判定

    ParseArguments(argc, argv, &config);

    if (strlen(config.searchStr) == 0) {
        printf("エラー: 検索文字列が指定されていません。\n");
        printf("使用方法: %s [/s] [/i 除外フォルダ] [/f utf8|cp932] <検索文字列> [<パス\\ワイルドカード>]\n", argv[0]);
        return 1;
    }

    // 設定確認表示
    printf("検索文字列  : %s\n", config.searchStr);
    printf("起点フォルダ: %s\n", config.baseDir);
    printf("検索パターン: %s\n", config.filePattern);
    printf("サブ再帰(/s): %s\n", config.recursive ? "ON" : "OFF");
    printf("ファイル指定: %s\n", config.encoding == ENC_UTF8 ? "UTF-8" : (config.encoding == ENC_CP932 ? "CP932" : "AUTO"));
    printf("除外フォルダ:\n");
    for (int i = 0; i < config.excludeCount; i++) {
        printf("  - %s\n", config.excludeDirs[i]);
    }
    printf("----------------------------------------\n");

    SearchDirectory(config.baseDir, &config);

    return 0;
}

void ParseArguments(int argc, char *argv[], SearchConfig *config) {
    bool searchStrSet = false;
    bool pathPatternSet = false;

    for (int i = 1; i < argc; i++) {
        if (strcmp(argv[i], "/s") == 0) {
            config->recursive = true;
        }
        else if (strcmp(argv[i], "/i") == 0) {
            if (i + 1 < argc) {
                if (config->excludeCount < MAX_EXCLUDES) {
                    strncpy(config->excludeDirs[config->excludeCount], argv[i + 1], MAX_PATH - 1);
                    config->excludeCount++;
                    i++; 
                }
            } else {
                printf("エラー: /i オプションの後には除外フォルダ名が必要です。\n");
                exit(1);
            }
        }
        // 【追加】/f オプションの解析
        else if (strcmp(argv[i], "/f") == 0) {
            if (i + 1 < argc) {
                if (_stricmp(argv[i + 1], "utf8") == 0) {
                    config->encoding = ENC_UTF8;
                } else if (_stricmp(argv[i + 1], "cp932") == 0) {
                    config->encoding = ENC_CP932;
                } else {
                    printf("エラー: /f の後は utf8 または cp932 を指定してください。\n");
                    exit(1);
                }
                i++; // 引数（utf8/cp932）をスキップ
            } else {
                printf("エラー: /f オプションの後には文字コード指定が必要です。\n");
                exit(1);
            }
        }
        else {
            if (!searchStrSet) {
                strncpy(config->searchStr, argv[i], sizeof(config->searchStr) - 1);
                searchStrSet = true;
            }
            else if (!pathPatternSet) {
                char fullPath[MAX_PATH];
                char *filePart = NULL;
                GetFullPathNameA(argv[i], MAX_PATH, fullPath, &filePart);

                if (filePart != NULL) {
                    strncpy(config->filePattern, filePart, sizeof(config->filePattern) - 1);
                    *(filePart - 1) = '\0';
                    strncpy(config->baseDir, fullPath, sizeof(config->baseDir) - 1);
                } else {
                    strncpy(config->filePattern, argv[i], sizeof(config->filePattern) - 1);
                }
                pathPatternSet = true;
            }
        }
    }
}

void SearchDirectory(const char *dirPath, const SearchConfig *config) {
    char searchPath[MAX_PATH];
    WIN32_FIND_DATAA findData;
    HANDLE hFind = INVALID_HANDLE_VALUE;

    snprintf(searchPath, sizeof(searchPath), "%s\\*", dirPath);
    hFind = FindFirstFileA(searchPath, &findData);

    if (hFind == INVALID_HANDLE_VALUE) return;

    int baseDirLen = (int)strlen(config->baseDir);

    do {
        if (strcmp(findData.cFileName, ".") == 0 || strcmp(findData.cFileName, "..") == 0) {
            continue;
        }

        char fullPath[MAX_PATH];
        snprintf(fullPath, sizeof(fullPath), "%s\\%s", dirPath, findData.cFileName);

        if (findData.dwFileAttributes & FILE_ATTRIBUTE_DIRECTORY) {
            if (isExcluded(findData.cFileName, config)) {
                continue; 
            }
            if (config->recursive) {
                SearchDirectory(fullPath, config);
            }
        } 
        else {
            if (PathMatchSpecA(findData.cFileName, config->filePattern)) {
                // 【変更】エンコード設定を渡す
                SearchInFile(fullPath, config->searchStr, baseDirLen, config->encoding);
            }
        }

    } while (FindNextFileA(hFind, &findData) != 0);

    FindClose(hFind);
}

void SearchInFile(const char *filePath, const char *searchStr, int baseDirLen, FileEncoding configEnc) {
    FILE *fp = fopen(filePath, "rb"); 
    if (fp == NULL) return;

    bool isUTF8 = false;
    size_t skipBytes = 0;

    // オプションが指定されているか、自動判定かで分岐
    if (configEnc == ENC_UTF8) {
        isUTF8 = true;
        // UTF-8固定でも、BOMがあればスキップする親切設計
        unsigned char bom[3];
        if (fread(bom, 1, 3, fp) == 3 && bom[0] == 0xEF && bom[1] == 0xBB && bom[2] == 0xBF) {
            skipBytes = 3;
        }
    } else if (configEnc == ENC_CP932) {
        isUTF8 = false;
    } else {
        // 【AUTO（自動判定）のロジック】
        unsigned char buf[4096];
        size_t readLen = fread(buf, 1, sizeof(buf), fp);
        if (readLen >= 3 && buf[0] == 0xEF && buf[1] == 0xBB && buf[2] == 0xBF) {
            isUTF8 = true;
            skipBytes = 3; 
        } else if (IsValidUTF8(buf, readLen)) {
            isUTF8 = true; 
        }
    }

    fseek(fp, (long)skipBytes, SEEK_SET);

    char line[1024];
    char displayLine[2048]; 
    int lineNumber = 1;

    const char *displayPath = filePath + baseDirLen;
    if (*displayPath == '\\') displayPath++;

    while (fgets(line, sizeof(line), fp) != NULL) {
        if (isUTF8) {
            ConvertUTF8toCP932(line, displayLine, sizeof(displayLine));
        } else {
            strncpy(displayLine, line, sizeof(displayLine) - 1);
            displayLine[sizeof(displayLine) - 1] = '\0';
        }

        if (strstr(displayLine, searchStr) != NULL) {
            printf("%s(%d): %s", displayPath, lineNumber, displayLine);
            if (displayLine[strlen(displayLine) - 1] != '\n') {
                printf("\n");
            }
        }
        lineNumber++;
    }
    fclose(fp);
}

bool IsValidUTF8(const unsigned char *str, size_t len) {
    size_t i = 0;
    while (i < len) {
        if (str[i] <= 0x7F) { i += 1; }
        else if ((str[i] & 0xE0) == 0xC0) { if (i + 1 >= len || (str[i+1] & 0xC0) != 0x80) return false; i += 2; }
        else if ((str[i] & 0xF0) == 0xE0) { if (i + 2 >= len || (str[i+1] & 0xC0) != 0x80 || (str[i+2] & 0xC0) != 0x80) return false; i += 3; }
        else if ((str[i] & 0xF8) == 0xF0) { if (i + 3 >= len || (str[i+1] & 0xC0) != 0x80 || (str[i+2] & 0xC0) != 0x80 || (str[i+3] & 0xC0) != 0x80) return false; i += 4; }
        else { return false; }
    }
    return true;
}

void ConvertUTF8toCP932(const char *utf8Src, char *cp932Dest, size_t destSize) {
    int wideSize = MultiByteToWideChar(CP_UTF8, 0, utf8Src, -1, NULL, 0);
    if (wideSize <= 0) {
        strncpy(cp932Dest, utf8Src, destSize); // 変換失敗時はそのまま
        return;
    }
    wchar_t *wideBuf = (wchar_t *)malloc(wideSize * sizeof(wchar_t));
    if (!wideBuf) return;
    MultiByteToWideChar(CP_UTF8, 0, utf8Src, -1, wideBuf, wideSize);
    WideCharToMultiByte(932, 0, wideBuf, -1, cp932Dest, (int)destSize, NULL, NULL);
    free(wideBuf);
}

bool isExcluded(const char *dirName, const SearchConfig *config) {
    for (int i = 0; i < config->excludeCount; i++) {
        if (_stricmp(dirName, config->excludeDirs[i]) == 0) {
            return true;
        }
    }
    return false;
}
