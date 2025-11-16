# file-transfer-project
// server.cpp : list / get 명령을 처리하는 TCP 파일 서버
// 빌드: cl /EHsc server.cpp ws2_32.lib

#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <windows.h>
#include <iostream>
#include <fstream>
#include <string>

#pragma comment(lib, "ws2_32.lib")

bool endsWith(const std::string& str, const std::string& suffix) {
    if (str.size() < suffix.size()) return false;
    return std::equal(suffix.rbegin(), suffix.rend(), str.rbegin());
}

// 폴더 내 txt, png 파일 목록 생성
std::string getFileList() {
    std::string result = "";
    WIN32_FIND_DATAA data;

    HANDLE hFind = FindFirstFileA("*.*", &data);
    if (hFind == INVALID_HANDLE_VALUE) return "";

    do {
        if (!(data.dwFileAttributes & FILE_ATTRIBUTE_DIRECTORY)) {
            std::string name = data.cFileName;

            if (endsWith(name, ".txt") || endsWith(name, ".png")) {
                result += name + "\n";
            }
        }
    } while (FindNextFileA(hFind, &data));

    FindClose(hFind);

    result += "END\n";  // 목록의 끝 표시
    return result;
}

int main() {
    WSADATA wsaData;
    WSAStartup(MAKEWORD(2, 2), &wsaData);

    SOCKET serverSock = socket(AF_INET, SOCK_STREAM, 0);

    sockaddr_in serverAddr{};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(9000);

    bind(serverSock, (SOCKADDR*)&serverAddr, sizeof(serverAddr));
    listen(serverSock, 1);

    std::cout << "[서버] 클라이언트 접속 대기...\n";

    sockaddr_in clientAddr{};
    int clientSize = sizeof(clientAddr);
    SOCKET clientSock = accept(serverSock, (SOCKADDR*)&clientAddr, &clientSize);

    std::cout << "[서버] 클라이언트 연결됨!\n";

    while (true) {
        char cmdBuf[256] = {};
        int recvBytes = recv(clientSock, cmdBuf, sizeof(cmdBuf) - 1, 0);
        if (recvBytes <= 0) break;

        std::string cmd(cmdBuf);

        // ====== list 명령 ======
        if (cmd == "list") {
            std::cout << "[서버] list 요청 받음\n";

            std::string list = getFileList();
            send(clientSock, list.c_str(), list.size(), 0);
        }

        // ====== get 명령 ======
        else if (cmd.rfind("get ", 0) == 0) {
            std::string filename = cmd.substr(4);
            std::cout << "[서버] get 요청: " << filename << "\n";

            std::ifstream file(filename, std::ios::binary);
            if (!file) {
                std::string err = "ERROR\n";
                send(clientSock, err.c_str(), err.size(), 0);
                continue;
            }

            char buffer[1024];
            while (!file.eof()) {
                file.read(buffer, sizeof(buffer));
                int bytes = file.gcount();
                if (bytes > 0) send(clientSock, buffer, bytes, 0);
            }

            std::cout << "[서버] 파일 전송 완료: " << filename << "\n";
        }

        else {
            std::cout << "[서버] 알 수 없는 명령: " << cmd << "\n";
        }
    }

    closesocket(clientSock);
    closesocket(serverSock);
    WSACleanup();
    return 0;
}
