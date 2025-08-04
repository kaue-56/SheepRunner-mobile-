        import pygame
import random
import sys
import math
import os

# Inicialização do Pygame
pygame.init()
pygame.mixer.init()

# Configurações de tela e responsividade
info = pygame.display.Info()
LARGURA_TELA, ALTURA_TELA = info.current_w, info.current_h
tela = pygame.display.set_mode((LARGURA_TELA, ALTURA_TELA), pygame.FULLSCREEN)
pygame.display.set_caption("Sheep Runner")

# Cores
BRANCO = (255, 255, 255)
PRETO = (0, 0, 0)
VERMELHO = (255, 0, 0)
VERDE = (0, 128, 0)
AZUL_CEU = (135, 206, 235)
OURO = (255, 215, 0)
MARROM = (139, 69, 19)
BRANCO_NUVEM = (240, 240, 240)

# Constantes do jogo
tamanho_base = min(LARGURA_TELA, ALTURA_TELA)
GRAVIDADE = tamanho_base * 0.002
FORCA_PULO = -tamanho_base * 0.035
Y_CHAO = ALTURA_TELA * 0.75
RAIO_OVELHA = int(tamanho_base * 0.045)
TAMANHO_PASSARO = int(tamanho_base * 0.03)
TAMANHO_PEDRA = int(tamanho_base * 0.03 * 1.5 * 0.7)
VELOCIDADE_JOGO = tamanho_base * 0.014
INCREMENTO_VELOCIDADE = 0.0045
DURACAO_PISCAR = 120
DESLOCAMENTO_ALTURA_PASSARO = 0.15
DISTANCIA_INTERNA_GRUPO = tamanho_base * 0.1
ATRASO_ANIMACAO = 100
ATRASO_ANIMACAO_PASSARO = 100
FATOR_PERSEGUICAO = 1.32
VELOCIDADE_NORMAL_MULTIPLIER = 1.3
VELOCIDADE_RAPIDO_MULTIPLIER = 1.5
VELOCIDADE_NUVEM_FATOR = 0.2

# Função para carregar sons com tratamento de erro
def carregar_som(nome_arquivo):
    """Carrega um arquivo de som, retornando um som vazio em caso de erro."""
    try:
        # Aparentemente, o caminho base pode estar errado, vamos usar um caminho relativo e depois absoluto
        caminho_base = os.path.dirname(__file__)
        caminho = os.path.join(caminho_base, "Assets", nome_arquivo)
        if not os.path.exists(caminho):
             caminho = os.path.join("/storage/emulated/0/Assets", nome_arquivo)
        
        som = pygame.mixer.Sound(caminho)
        som.set_volume(0.5)
        return som
    except Exception as e:
        print(f"Aviso: Não foi possível carregar o som {nome_arquivo}: {e}")
        return pygame.mixer.Sound(buffer=bytearray(0))

# Função para carregar imagens com tratamento de erro e otimização
def carregar_imagem(nome_arquivo, tamanho=None, flip=False, escala=1.0):
    """Carrega, otimiza e redimensiona uma imagem."""
    try:
        # Tenta o caminho absoluto
        caminho = os.path.join("/storage/emulated/0/Assets", nome_arquivo)
        
        # Se não encontrar, tenta o caminho relativo
        if not os.path.exists(caminho):
            caminho_base = os.path.dirname(__file__)
            caminho = os.path.join(caminho_base, "Assets", nome_arquivo)

        img = pygame.image.load(caminho).convert_alpha()
        
        if flip:
            img = pygame.transform.flip(img, True, False)
        
        if escala != 1.0:
            w, h = img.get_size()
            img = pygame.transform.scale(img, (int(w * escala), int(h * escala)))
        
        if tamanho:
            img = pygame.transform.scale(img, tamanho)
        
        return img
    except Exception as e:
        print(f"Erro ao carregar imagem {nome_arquivo}: {e}")
        # Retorna uma superfície de fallback para evitar o erro.
        fallback_surface = pygame.Surface((1, 1), pygame.SRCALPHA)
        return fallback_surface

# Carregar assets
img_chao = carregar_imagem("chao.png", (LARGURA_TELA, int(ALTURA_TELA * 0.25)))
img_pedra1 = carregar_imagem("Rock1.png", (int(TAMANHO_PEDRA * 2), int(TAMANHO_PEDRA * 2)))
img_pedra2 = carregar_imagem("Rock2.png", (int(TAMANHO_PEDRA * 2), int(TAMANHO_PEDRA * 2)))
img_nuvem = carregar_imagem("cloud.png", (int(LARGURA_TELA * 0.15), int(ALTURA_TELA * 0.1)))

# Função para carregar animações
def carregar_animacao(prefixo, contagem, escala=1.0, flip=False):
    """Carrega uma sequência de frames de animação."""
    frames = []
    for i in range(1, contagem + 1):
        frame = carregar_imagem(f"{prefixo}{i}.png", escala=escala, flip=flip)
        if frame and frame.get_size() != (1,1): # Verifica se não é a superfície de fallback
            frames.append(frame)
    # Garante que a lista de frames não esteja vazia
    if not frames:
        # Cria um fallback de superfície colorida para evitar IndexErrors
        fallback_surface = pygame.Surface((RAIO_OVELHA * 2, RAIO_OVELHA * 2), pygame.SRCALPHA)
        pygame.draw.circle(fallback_surface, VERMELHO, (RAIO_OVELHA, RAIO_OVELHA), RAIO_OVELHA)
        frames.append(fallback_surface)
        print(f"Aviso: Nenhuma animação carregada para '{prefixo}', usando fallback.")
    return frames

# Carregar frames com aumento de 50% para a ovelha (escala 1.5)
frames_correndo = carregar_animacao("sheep/run", 6, 1.5)
frames_pulo = carregar_animacao("sheep/jump", 8, 1.5)
frames_morte = carregar_animacao("sheep/death", 8, 1.5)
frames_passaro = carregar_animacao("Bird/fly", 6, 1.0, True)

# A verificação de fallback foi movida para a função carregar_animacao,
# então não é mais necessária aqui.
# if not frames_correndo: ...

# Carregar sons
sons = {
    'pulo': carregar_som("pulo.wav"),
    'colisao': carregar_som("colisao.wav"),
    'pontuacao': carregar_som("score.wav"),
    'recorde': carregar_som("record.wav"),
    'falha': carregar_som("fail.wav"),
    'menu': carregar_som("menu.wav")
}

class Ovelha:
    """Representa o jogador (a ovelha)."""
    def __init__(self):
        self.raio = RAIO_OVELHA
        self.frames_correndo = frames_correndo
        self.frames_pulo = frames_pulo
        self.frames_morte = frames_morte
        self.resetar()
    
    def resetar(self):
        """Reinicia o estado do jogador para um novo jogo."""
        self.x = LARGURA_TELA * 0.15
        self.y = Y_CHAO - self.raio
        self.vel_y = 0
        self.contador_pulo = 0
        self.esta_morta = False
        self.frame_atual = 0
        self.temporizador_animacao = 0
        self.ultima_atualizacao = pygame.time.get_ticks()
        self.animacao_atual = self.frames_correndo
        self.animacao_morte_completa = False
    
    def pular(self):
        """Faz a ovelha pular se não estiver morta."""
        if not self.esta_morta and self.contador_pulo < 2:
            self.vel_y = FORCA_PULO
            self.contador_pulo += 1
            self.animacao_atual = self.frames_pulo
            self.frame_atual = 0
            self.temporizador_animacao = 0
            if sons['pulo']:
                sons['pulo'].play()
    
    def iniciar_morte(self):
        """Inicia a animação de morte da ovelha."""
        self.esta_morta = True
        self.vel_y = -30
        self.animacao_atual = self.frames_morte
        self.frame_atual = 0
        self.temporizador_animacao = 0
        self.animacao_morte_completa = False
        if sons['colisao']:
            sons['colisao'].play()
    
    def atualizar(self):
        """Atualiza a posição e animação do jogador."""
        tempo_atual = pygame.time.get_ticks()
        dt = tempo_atual - self.ultima_atualizacao
        self.ultima_atualizacao = tempo_atual
        
        # Atualizar animação
        self.temporizador_animacao += dt
        # Corrigido: Verifique se a lista de animação não está vazia antes de usar len()
        if self.animacao_atual and self.temporizador_animacao >= ATRASO_ANIMACAO:
            self.temporizador_animacao = 0
            if not self.esta_morta or not self.animacao_morte_completa:
                self.frame_atual += 1
                if self.frame_atual >= len(self.animacao_atual):
                    if self.esta_morta:
                        self.frame_atual = len(self.animacao_atual) - 1
                        self.animacao_morte_completa = True
                    else:
                        self.frame_atual = 0
        
        # Aplicar gravidade
        self.vel_y += GRAVIDADE
        self.y += self.vel_y
        
        if not self.esta_morta:
            if self.y >= Y_CHAO - self.raio:
                self.y = Y_CHAO - self.raio
                self.vel_y = 0
                self.contador_pulo = 0
                self.animacao_atual = self.frames_correndo
        else:
            if self.y > ALTURA_TELA + 100:
                return True
        return False
    
    def desenhar(self, superficie):
        """Desenha o jogador na tela."""
        # Corrigido: Acessa o frame apenas se a lista de animação não estiver vazia.
        if self.animacao_atual:
            frame_idx = min(self.frame_atual, len(self.animacao_atual) - 1)
            frame = self.animacao_atual[frame_idx]
            retangulo = frame.get_rect(midbottom=(self.x, self.y + self.raio))
            superficie.blit(frame, retangulo)
        else:
            # Fallback para desenho, caso a animação falhe totalmente
            pygame.draw.circle(superficie, VERMELHO, (int(self.x), int(self.y + self.raio)), self.raio)


class Chao:
    """Representa o chão que se move."""
    def __init__(self):
        self.y = Y_CHAO
        self.largura = LARGURA_TELA
        self.x1 = 0
        self.x2 = self.largura
    
    def atualizar(self, velocidade):
        """Atualiza a posição do chão para criar um efeito de rolagem."""
        self.x1 -= velocidade
        self.x2 -= velocidade
        
        if self.x1 + self.largura < 0:
            self.x1 = self.x2 + self.largura
        if self.x2 + self.largura < 0:
            self.x2 = self.x1 + self.largura
    
    def desenhar(self, superficie):
        """Desenha o chão na tela."""
        if img_chao and img_chao.get_size() != (1,1):
            superficie.blit(img_chao, (self.x1, self.y))
            superficie.blit(img_chao, (self.x2, self.y))
        else:
            pygame.draw.rect(superficie, MARROM, (self.x1, self.y, self.largura, ALTURA_TELA - Y_CHAO))

class Nuvem:
    """Representa as nuvens no céu."""
    def __init__(self):
        self.resetar()
    
    def resetar(self):
        """Reinicia a nuvem em uma nova posição aleatória."""
        self.x = LARGURA_TELA + random.randint(0, LARGURA_TELA)
        self.y = random.randint(0, int(ALTURA_TELA * 0.5))
        self.velocidade_individual = random.uniform(0.5, 2.0)
        self.tamanho = random.uniform(0.5, 1.2)
        self.largura = int(LARGURA_TELA * 0.15 * self.tamanho)
        self.altura = int(ALTURA_TELA * 0.1 * self.tamanho)
