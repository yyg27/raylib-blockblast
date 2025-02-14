#include "raylib.h"
#include <string.h>
#include <stdio.h>

// Global tanımlamalar ve yapı tanımları burada
#define MAX_SIZE 4
#define GRID_SIZE 8
#define CELL_SIZE 40
#define PIECE_COUNT 25
#define HIGH_SCORE_FILE "highscore.txt"

// Ses dosyası
Sound explosionSound;
Sound gameOverSound;
Sound placedsound;
bool isAnimating = false;
int animationFrames = 0;
const int animationDuration = 30; // Patlama animasyonu süresi (30 frame)
bool explosionGrid[GRID_SIZE][GRID_SIZE] = {false}; // Patlama animasyonunun yapılacağı hücreler
bool gameOverSoundPlayed = false; // Oyun bittiğinde sesin bir kez çalınıp çalınmadığını kontrol eder
int LoadHighScore();
void SaveHighScore(int score);
int highScore = 0;

typedef struct {
    int sizeX, sizeY;
    int shape[MAX_SIZE][MAX_SIZE];
    Color color;
    bool used; // Parçanın kullanılıp kullanılmadığını belirten bayrak
} Piece;

typedef struct {
    int pieceIndex;
    Color color;
} GridCell;

typedef enum GameScreen { LOGO = 0, TITLE, GAMEPLAY, ENDING } GameScreen;

// Parçaları tanımlayalım
Piece pieces[PIECE_COUNT] = {
    {2, 3, {{1, 0 }, {1, 0 }, {1, 1}}, YELLOW, false},   // Sarı L
    {2, 3, {{0, 1}, {0, 1}, {1, 1}}, GOLD, false},       // Altın Ters L
    {3, 2, {{0, 1, 0}, {1, 1, 1}}, ORANGE, false},       // Turuncu T
    {3, 2, {{1, 1, 1}, {0, 1, 0}}, PINK, false},         // Pembe T
    {2, 2, {{1, 1}, {1, 1}}, RED, false},                // Kırmızı Kare
    {3, 1, {{1, 1, 1}}, GREEN, false},                   // Yeşil Çubuk
    {4, 1, {{1, 1, 1, 1}}, LIME, false},                  // Yeşil Çubuk
    {3, 2, {{1, 1, 1}, {1, 0, 0}}, GOLD, false},         // Limon J
    {3, 2, {{1, 1, 1}, {0, 0, 1}}, GREEN, false},         // Mavi J Ters
    {3, 2, {{1, 1, 0}, {0, 1, 1}}, PURPLE, false},     // Koyu Mavi Z
    {3, 2, {{0, 1, 1}, {1, 1, 0}}, VIOLET, false},       // Sarı S
    {2, 3, {{0, 1}, {1, 1}, {0, 1}}, PURPLE, false},       // Altın T Ters
    {1, 3, {{1}, {1}, {1}}, YELLOW, false},              // Turuncu Çubuk
    {1, 4, {{1}, {1}, {1}, {1}}, GOLD, false},         // Turuncu Çubuk
    {1, 2, {{1}, {1}}, ORANGE, false},                     // Pembe Çubuk
    {2, 1, {{1, 1}}, PINK, false},                        // Kırmızı Çift
    {2, 2, {{1, 0}, {1, 1}}, RED, false},              // Yeşil Köşe
    {3, 2, {{1, 0, 0}, {1, 1, 1}}, GREEN, false},         // Limon L
    {2, 3, {{1, 1}, {1, 0}, {1, 0}}, LIME, false},    // Gökyüzü Mavi Ters L
    {2, 2, {{1, 1}, {1, 0}}, BLUE, false},               // Mavi Köşe
    {2, 3, {{1, 1}, {0, 1}, {0, 1}}, GREEN, false},     // Mor Ters L
    {2, 2, {{1, 1}, {0, 1}}, VIOLET, false},             // Menekşe Köşe
    {2, 2, {{0, 1}, {1, 1}}, PURPLE, false},              // Kahverengi Köşe
    {3, 2, {{0, 0, 1}, {1, 1, 1}}, GREEN, false},        // Yeşil Ters L
    {2, 3, {{1, 0}, {1, 1}, {1, 0}}, VIOLET, false}         // Kırmızı Ters T
};

// Fonksiyon prototipleri
void DrawGameGrid(GridCell grid[GRID_SIZE][GRID_SIZE], int offsetX, int offsetY);
void DrawPiece(Piece p, int posX, int posY, int cellSize, bool isDragging);
void DrawPiecePool(Piece pool[3], int offsetX, int offsetY, int draggedPiece);
void InitializePiecePool(Piece pool[3]);
void UpdatePiecePool(Piece pool[3], int *piecesUsedCount);
void ClearFullLines(GridCell grid[GRID_SIZE][GRID_SIZE], int *score);
void PlacePieceIfPossible(GridCell grid[GRID_SIZE][GRID_SIZE], Piece piecePool[3], int *piecesUsedCount, int selectedPieceIndex, int mouseX, int mouseY, int *score);
bool CanPlacePiece(GridCell grid[GRID_SIZE][GRID_SIZE], Piece *p, int startX, int startY);
bool IsGameOver(GridCell grid[GRID_SIZE][GRID_SIZE], Piece piecePool[3]);
bool DrawButton(int posX, int posY, int width, int height, const char *text);
void FillRandomCells(GridCell grid[GRID_SIZE][GRID_SIZE], int count, Color color);
void DrawExplosion(int x, int y);

// Fonksiyon tanımlamaları
void DrawGameGrid(GridCell grid[GRID_SIZE][GRID_SIZE], int offsetX, int offsetY) {
    for (int i = 0; i < GRID_SIZE; i++) {
        for (int j = 0; j < GRID_SIZE; j++) {
            Color color = (grid[i][j].pieceIndex == 0) ? LIGHTGRAY : grid[i][j].color;
            DrawRectangle(offsetX + j * CELL_SIZE, offsetY + i * CELL_SIZE, CELL_SIZE - 2, CELL_SIZE - 2, color);
        }
    }
}

void DrawPiece(Piece p, int posX, int posY, int cellSize, bool isDragging) {
    if (!p.used) {
        for (int i = 0; i < p.sizeY; i++) {
            for (int j = 0; j < p.sizeX; j++) {
                if (p.shape[i][j] == 1) {
                    DrawRectangle(posX + j * cellSize, posY + i * cellSize, cellSize - 2, cellSize - 2, isDragging ? Fade(p.color, 0.5f) : p.color);
                }
            }
        }
    }
}

void DrawPiecePool(Piece pool[3], int offsetX, int offsetY, int draggedPiece) {
    for (int i = 0; i < 3; i++) {
        if (i != draggedPiece) {
            DrawPiece(pool[i], offsetX, offsetY + i * (MAX_SIZE * CELL_SIZE + 10), CELL_SIZE, false);
        }
    }
}

void InitializePiecePool(Piece pool[3]) {
    for (int i = 0; i < 3; i++) {
        int randIndex = GetRandomValue(0, PIECE_COUNT - 1);
        pool[i] = pieces[randIndex];
        pool[i].used = false;
    }
}

void UpdatePiecePool(Piece pool[3], int *piecesUsedCount) {
    // Eğer üç parça da kullanıldıysa, yeni parça havuzu oluştur
    if (*piecesUsedCount == 3) {
        InitializePiecePool(pool);
        *piecesUsedCount = 0;
    }
}

void ClearFullLines(GridCell grid[GRID_SIZE][GRID_SIZE], int *score) {
    for (int i = 0; i < GRID_SIZE; i++) {
        bool rowFull = true;
        bool colFull = true;

        for (int j = 0; j < GRID_SIZE; j++) {
            if (grid[i][j].pieceIndex == 0) {
                rowFull = false;
            }
            if (grid[j][i].pieceIndex == 0) {
                colFull = false;
            }
        }

        if (rowFull) {
            for (int j = 0; j < GRID_SIZE; j++) {
                grid[i][j].pieceIndex = 0;
                grid[i][j].color = LIGHTGRAY;
                explosionGrid[i][j] = true; // Patlama animasyonu için işaretle
            }
            *score += 100;  // Tam satır temizlendiğinde 100 puan ekle
            PlaySound(explosionSound); // Patlama sesini çal
            isAnimating = true; // Animasyonu başlat
            animationFrames = 0;
        }

        if (colFull) {
            for (int j = 0; j < GRID_SIZE; j++) {
                grid[j][i].pieceIndex = 0;
                grid[j][i].color = LIGHTGRAY;
                explosionGrid[j][i] = true; // Patlama animasyonu için işaretle
            }
            *score += 100;  // Tam sütun temizlendiğinde 100 puan ekle
            PlaySound(explosionSound); // Patlama sesini çal
            isAnimating = true; // Animasyonu başlat
            animationFrames = 0;
        }
    }
}

void PlacePieceIfPossible(GridCell grid[GRID_SIZE][GRID_SIZE], Piece piecePool[3], int *piecesUsedCount, int selectedPieceIndex, int mouseX, int mouseY, int *score) {
    Piece *p = &piecePool[selectedPieceIndex];

    // Parçanın dolu hücrelerinin başlangıcını bul
    int firstX = -1, firstY = -1;
    for (int y = 0; y < p->sizeY; y++) {
        for (int x = 0; x < p->sizeX; x++) {
            if (p->shape[y][x] == 1) {
                if (firstX == -1 || x < firstX) firstX = x;
                if (firstY == -1 || y < firstY) firstY = y;
            }
        }
    }

    // Mouse pozisyonundan grid üzerindeki başlangıç hücresini hesapla
    int startX = (mouseX - 80) / CELL_SIZE - firstX;
    int startY = (mouseY - 80) / CELL_SIZE - firstY;

    // Parçanın grid üzerinde yerleştirilip yerleştirilemeyeceğini kontrol et
    bool canPlace = true;
    for (int y = 0; y < p->sizeY; y++) {
        for (int x = 0; x < p->sizeX; x++) {
            if (p->shape[y][x] == 1) {
                int gridX = startX + x;
                int gridY = startY + y;

                // Grid sınırları içinde ve hücre boşsa kontrol et
                if (gridX < 0 || gridX >= GRID_SIZE || gridY < 0 || gridY >= GRID_SIZE || grid[gridY][gridX].pieceIndex != 0) {
                    canPlace = false;
                    break;
                }
            }
        }
        if (!canPlace) break;
    }

    // Parçayı yerleştir
    if (canPlace) {
        for (int y = 0; y < p->sizeY; y++) {
            for (int x = 0; x < p->sizeX; x++) {
                if (p->shape[y][x] == 1) {
                    int gridX = startX + x;
                    int gridY = startY + y;
                    grid[gridY][gridX].pieceIndex = selectedPieceIndex + 1;
                    grid[gridY][gridX].color = p->color;
                    PlaySound(placedsound);
                }
            }
        }
        
        // Parça yerleştirildiğinde puan ekle
        int pieceScore = 0;
        for (int y = 0; y < p->sizeY; y++) {
            for (int x = 0; x < p->sizeX; x++) {
                if (p->shape[y][x] == 1) {
                    pieceScore += 10;  // Her dolu hücre için 10 puan ekle
                }
            }
        }
        *score += pieceScore;
        
        ClearFullLines(grid, score); // Dolmuş satır ve sütunları temizle ve skoru güncelle

        // Parçanın kullanıldığını işaretle
        p->used = true;
        
        // Kullanılan parça sayısını artır ve gerekirse parça havuzunu yenile
        (*piecesUsedCount)++;
        UpdatePiecePool(piecePool, piecesUsedCount);
    }
}

bool CanPlacePiece(GridCell grid[GRID_SIZE][GRID_SIZE], Piece *p, int startX, int startY) {
    for (int y = 0; y < p->sizeY; y++) {
        for (int x = 0; x < p->sizeX; x++) {
            if (p->shape[y][x] == 1) {
                int gridX = startX + x;
                int gridY = startY + y;

                // Grid sınırları içinde ve hücre boşsa kontrol et
                if (gridX < 0 || gridX >= GRID_SIZE || gridY < 0 || gridY >= GRID_SIZE || grid[gridY][gridX].pieceIndex != 0) {
                    return false;
                }
            }
        }
    }
    return true;
}

bool IsGameOver(GridCell grid[GRID_SIZE][GRID_SIZE], Piece piecePool[3]) {
    for (int i = 0; i < 3; i++) {
        Piece *p = &piecePool[i];
        if (!p->used) {
            for (int y = 0; y < GRID_SIZE; y++) {
                for (int x = 0; x < GRID_SIZE; x++) {
                    if (CanPlacePiece(grid, p, x, y)) {
                        return false; // Yerleştirilebilir bir yer varsa oyun devam ediyor
                    }
                }
            }
        }
    }
    return true; // Hiçbir parça yerleştirilemiyorsa oyun bitti
}

bool DrawButton(int posX, int posY, int width, int height, const char *text) {
    int mouseX = GetMouseX();
    int mouseY = GetMouseY();
    bool isHovered = mouseX >= posX && mouseX <= posX + width && mouseY >= posY && mouseY <= posY + height;
    bool isClicked = isHovered && IsMouseButtonPressed(MOUSE_BUTTON_LEFT);

    Color buttonColor = isHovered ? DARKGRAY : GRAY;

    DrawRectangle(posX, posY, width, height, buttonColor);
    DrawText(text, posX + width / 2 - MeasureText(text, 20) / 2, posY + height / 2 - 10, 20, RAYWHITE);

    return isClicked;
}

void FillRandomCells(GridCell grid[GRID_SIZE][GRID_SIZE], int count, Color color) {
    int filled = 0;
    while (filled < count) {
        int x = GetRandomValue(0, GRID_SIZE - 1);
        int y = GetRandomValue(0, GRID_SIZE - 1);

        if (grid[y][x].pieceIndex == 0) { // Hücre boşsa doldur
            grid[y][x].pieceIndex = 1; // Rastgele bir index veriyoruz
            grid[y][x].color = color;
            filled++;
        }
    }
}

void DrawExplosion(int x, int y) {
    DrawRectangleLines(x, y, CELL_SIZE, CELL_SIZE, ORANGE);
}

int LoadHighScore() {
    FILE *file = fopen(HIGH_SCORE_FILE, "r");
    int highScore = 0;
    if (file != NULL) {
        fscanf(file, "%d", &highScore);
        fclose(file);
    }
    return highScore;
}

void SaveHighScore(int score) {
    FILE *file = fopen(HIGH_SCORE_FILE, "w");
    if (file != NULL) {
        fprintf(file, "%d", score);
        fclose(file);
    }
}

int main(void) {
    const int screenWidth = 800;
    const int screenHeight = 600;
    InitWindow(screenWidth, screenHeight, "BLOCK BLAST");

    InitAudioDevice(); // Ses cihazını başlat
    explosionSound = LoadSound("ses2.mp3"); // Ses dosyasını yükle
    gameOverSound = LoadSound("gameover.mp3"); // Game over ses dosyasını yükle
    placedsound = LoadSound("ses.mp3");
    GameScreen currentScreen = LOGO;

    int framesCounter = 0;          
    int piecesUsedCount = 0; 
    GridCell grid[GRID_SIZE][GRID_SIZE] = {0};
    Piece piecePool[3];
    InitializePiecePool(piecePool);

    FillRandomCells(grid, 15, SKYBLUE); // Grid üzerine rastgele 20 hücreyi doldur

    bool isDragging = false;
    int draggedPiece = -1;
    int dragOffsetX = 0;
    int dragOffsetY = 0;

    int score = 0;
    highScore = LoadHighScore();

    SetTargetFPS(60);               

    while (!WindowShouldClose())    
    {
        switch(currentScreen)
        {
            case LOGO:
                framesCounter++;
                if (framesCounter > 120) {
                    currentScreen = TITLE;
                }
                break;
            case TITLE:
                if (IsKeyPressed(KEY_ENTER) || IsGestureDetected(GESTURE_TAP)) {
                    currentScreen = GAMEPLAY;
                    framesCounter = 0;
                    memset(grid, 0, sizeof(grid));
                    memset(explosionGrid, 0, sizeof(explosionGrid)); // Patlama gridini sıfırla
                    InitializePiecePool(piecePool);
                    FillRandomCells(grid, 15, SKYBLUE); // Grid üzerine rastgele 20 hücreyi doldur
                    score = 0;
                    piecesUsedCount = 0;
                    gameOverSoundPlayed = false; // Oyun sesini sıfırla
                }
                break;
            case GAMEPLAY:
                int mouseX = GetMouseX();
                int mouseY = GetMouseY();

                if (IsMouseButtonPressed(MOUSE_BUTTON_LEFT)) {
                    for (int i = 0; i < 3; i++) {
                        int pieceX = 500;
                        int pieceY = 100 + i * (MAX_SIZE * CELL_SIZE + 10);
                        if (CheckCollisionPointRec(GetMousePosition(), (Rectangle){ pieceX, pieceY, piecePool[i].sizeX * CELL_SIZE, piecePool[i].sizeY * CELL_SIZE })) {
                            int localX = (mouseX - pieceX) / CELL_SIZE;
                            int localY = (mouseY - pieceY) / CELL_SIZE;
                            if (piecePool[i].shape[localY][localX] == 1) {
                                isDragging = true;
                                draggedPiece = i;
                                dragOffsetX = mouseX - pieceX;
                                dragOffsetY = mouseY - pieceY;
                                break;
                            }
                        }
                    }
                }

                if (IsMouseButtonReleased(MOUSE_BUTTON_LEFT) && isDragging) {
                    PlacePieceIfPossible(grid, piecePool, &piecesUsedCount, draggedPiece, mouseX - dragOffsetX, mouseY - dragOffsetY, &score);
                    isDragging = false;
                    draggedPiece = -1;
                }

                if (IsGameOver(grid, piecePool)) {
                    currentScreen = ENDING;
                    if (score > highScore) {
                        highScore = score;
                        SaveHighScore(highScore);
                    }
                }
                
                if (DrawButton(screenWidth - 100, screenHeight - 50, 90, 40, "Replay")) {
                    currentScreen = TITLE;
                }

                DrawGameGrid(grid, 100, 100);

                if (isDragging && draggedPiece != -1) {
                    DrawPiecePool(piecePool, 500, 100, draggedPiece); // Sürüklenen parçayı atlayarak çiz
                    DrawPiece(piecePool[draggedPiece], mouseX - dragOffsetX, mouseY - dragOffsetY, CELL_SIZE, true);
                } else {
                    DrawPiecePool(piecePool, 500, 100, -1); // Tüm parçaları çiz
                }

                if (isAnimating) {
                    for (int i = 0; i < GRID_SIZE; i++) {
                        for (int j = 0; j < GRID_SIZE; j++) {
                            if (explosionGrid[i][j] && animationFrames < animationDuration) {
                                DrawExplosion(100 + j * CELL_SIZE, 100 + i * CELL_SIZE);
                            }
                        }
                    }
                    animationFrames++;
                    if (animationFrames >= animationDuration) {
                        isAnimating = false;
                        memset(explosionGrid, 0, sizeof(explosionGrid)); // Patlama gridini sıfırla
                    }
                }

                DrawText(TextFormat("SCORE: %d", score), 200, 50, 20, DARKGRAY);
                DrawText(TextFormat("HIGH SCORE: %d", highScore), 380, 50, 20, DARKBLUE);
                break;
            case ENDING:
                if (!gameOverSoundPlayed) {
                    PlaySound(gameOverSound);
                    gameOverSoundPlayed = true;
                }
   
                DrawRectangle(0, 0, screenWidth, screenHeight, BLUE);
                DrawText("GAME OVER", 200, 220, 40, RED);
                DrawText(TextFormat("FINAL SCORE: %d", score), 200, 300, 30, DARKBLUE);
                DrawText(TextFormat("HIGH SCORE: %d", highScore), 200, 340, 30, DARKBLUE);
                DrawText("PRESS ENTER or TAP to RETURN to TITLE SCREEN", 120, 400, 20, DARKBLUE);
                if (IsKeyPressed(KEY_ENTER) || IsGestureDetected(GESTURE_TAP)) {
                    currentScreen = TITLE;
                    framesCounter = 0;
                }
                break;
            default:
                break;
        }
        BeginDrawing();
        ClearBackground(RAYWHITE);

        switch(currentScreen)
        {
            case LOGO:
                DrawText("START SCREEN", 20, 20, 10, LIGHTGRAY);
                DrawText("WELCOME TO OUR GAME", 150, 220, 40, BLUE);
                break;
            case TITLE:
                DrawRectangle(0, 0, screenWidth, screenHeight, SKYBLUE);
                const char *titleText = "BLOCK BLAST";
                int textWidth = MeasureText(titleText, 80);
                DrawText(titleText, (screenWidth - textWidth) / 2, 150, 80, YELLOW);
                DrawText("PRESS ENTER or TAP to START GAME", 220, 500, 15, GRAY);
                Piece tetromino = {3, 3, {{1, 0, 0}, {1, 1, 0}, {1, 0, 0}}, RED, false};
                Piece tetromino2 = {2, 3, {{1, 1}, {0, 1}, {0, 1}}, PURPLE, false};
                Piece tetromino3 = {2, 2, {{0, 1}, {1, 1}}, GREEN, false};
                Piece tetromino4 = {1, 4, {{1}, {1}, {1}, {1}}, ORANGE, false};
                DrawPiece(tetromino, 300, 300, CELL_SIZE, false);
                DrawPiece(tetromino2, 340, 300, CELL_SIZE, false);
                DrawPiece(tetromino3, 300, 380, CELL_SIZE, false);
                DrawPiece(tetromino4, 420, 300, CELL_SIZE, false);
                break;
            case GAMEPLAY:
                DrawGameGrid(grid, 100, 100);

                if (isDragging && draggedPiece != -1) {
                    int mouseX = GetMouseX();
                    int mouseY = GetMouseY();
                    DrawPiecePool(piecePool, 500, 100, draggedPiece); // Sürüklenen parçayı atlayarak çiz
                    DrawPiece(piecePool[draggedPiece], mouseX - dragOffsetX, mouseY - dragOffsetY, CELL_SIZE, true);
                } else {
                    DrawPiecePool(piecePool, 500, 100, -1); // Tüm parçaları çiz
                }

                if (isAnimating) {
                    for (int i = 0; i < GRID_SIZE; i++) {
                        for (int j = 0; j < GRID_SIZE; j++) {
                            if (explosionGrid[i][j] && animationFrames < animationDuration) {
                                DrawExplosion(100 + j * CELL_SIZE, 100 + i * CELL_SIZE);
                            }
                        }
                    }
                    animationFrames++;
                    if (animationFrames >= animationDuration) {
                        isAnimating = false;
                        memset(explosionGrid, 0, sizeof(explosionGrid)); // Patlama gridini sıfırla
                    }
                }

                DrawText(TextFormat("SCORE: %d", score), 200, 50, 20, DARKGRAY);
                DrawText(TextFormat("HIGH SCORE: %d", highScore), 380, 50, 20, DARKBLUE);
                break;
            case ENDING:
                if (!gameOverSoundPlayed) {
                    PlaySound(gameOverSound);
                    gameOverSoundPlayed = true;
                }
   
                DrawRectangle(0, 0, screenWidth, screenHeight, BLUE);
                DrawText("GAME OVER", 200, 220, 40, RED);
                DrawText(TextFormat("FINAL SCORE: %d", score), 200, 300, 30, DARKBLUE);
                DrawText(TextFormat("HIGH SCORE: %d", highScore), 200, 340, 30, DARKBLUE);
                DrawText("PRESS ENTER or TAP to RETURN to TITLE SCREEN", 120, 400, 20, DARKBLUE);
                if (IsKeyPressed(KEY_ENTER) || IsGestureDetected(GESTURE_TAP)) {
                    currentScreen = TITLE;
                    framesCounter = 0;
                }
                break;
            default:
                break;
        }

        EndDrawing();
    }

    CloseAudioDevice(); // Ses cihazını kapat
    UnloadSound(explosionSound); // Ses dosyasını serbest bırak
    UnloadSound(gameOverSound); // Game over ses dosyasını serbest bırak
    UnloadSound(placedsound);
    CloseWindow();       

    return 0;
}
