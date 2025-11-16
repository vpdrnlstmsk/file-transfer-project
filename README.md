# file-transfer-project
// server.cpp : list + get 기능 포함 서버 코드

#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <iostream>
#include <fstream>
#include <vector>
#include <string>
#include <filesystem>
namespace fs = std::filesystem;

#pragma comment(lib, "ws2_32.lib")

std::vector<std::string> getFileList() {
    std::vector<std::string> list;
    for (auto& entry : fs::directory_iterator(".")) {
        if (entry.path().extension() == ".txt" ||
            entry.path().extension() == ".png") {
            list.push_back(entry.path().filename().string());
        }
    }
    return list;
}

int main() {
    WSADATA wsaData;
    WSAStartup(MAKEWORD(2, 2), &wsaData);

    SOCKET serverSock = socket(AF_INET, SOCK_STREAM, 0);

    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(9000);

    bind(serverSock, (SOCKADDR*)&serverAddr, sizeof(serverAddr));
    listen(serverSock, 1);

    std::cout << "[서버] 클라이언트 접속 대기 중...\n";

    sockaddr_in clientAddr;
    int clientSize = sizeof(clientAddr);
    SOCKET clientSock = accept(serverSock, (SOCKADDR*)&clientAddr, &clientSize);

    std::cout << "[서버] 클라이언트 연결됨!\n";

    char cmdBuffer[256];

    while (true) {
        memset(cmdBuffer, 0, sizeof(cmdBuffer));
        int recvLen = recv(clientSock, cmdBuffer, sizeof(cmdBuffer), 0);
        if (recvLen <= 0) break;

        std::string command = cmdBuffer;
        std::cout << "[서버] 받은 명령: " << command << "\n";

        // ---------------------------
        // LIST 처리
        // ---------------------------
        if (command == "list") {
            auto files = getFileList();

            std::string response;
            for (auto& f : files) response += f + "\n";

            if (response.empty())
                response = "No files found.\n";

            send(clientSock, response.c_str(), response.size(), 0);
        }

        // ---------------------------
        // GET 처리
        // ---------------------------
        else if (command.rfind("get ", 0) == 0) {
            std::string filename = command.substr(4);

            std::ifstream file(filename, std::ios::binary);
            if (!file) {
                std::string msg = "[서버] 파일 없음\n";
                send(clientSock, msg.c_str(), msg.size(), 0);
                continue;
            }

            char buffer[1024];
            while (!file.eof()) {
                file.read(buffer, sizeof(buffer));
                int bytesRead = file.gcount();
                if (bytesRead > 0)
                    send(clientSock, buffer, bytesRead, 0);
            }
            std::cout << "[서버] 파일 전송 완료\n";
        }
    }

    closesocket(clientSock);
    closesocket(serverSock);
    WSACleanup();
    return 0;
}
