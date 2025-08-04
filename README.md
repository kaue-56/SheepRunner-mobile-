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
        # usando caminho absoluto
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
    
    def atualizar(self, velocidade_jogo):
        """Atualiza a posição da nuvem, acelerando com o jogo."""
        self.x -= velocidade_jogo * VELOCIDADE_NUVEM_FATOR
        if self.x < -self.largura:
            self.resetar()
    
    def desenhar(self, superficie):
        """Desenha a nuvem na tela."""
        if img_nuvem and img_nuvem.get_size() != (1,1):
            nuvem_redimensionada = pygame.transform.scale(img_nuvem, (self.largura, self.altura))
            superficie.blit(nuvem_redimensionada, (self.x, self.y))
        else:
            pygame.draw.ellipse(superficie, BRANCO_NUVEM, (self.x, self.y, self.largura, self.altura))

class Passaro:
    """Representa o pássaro, um obstáculo voador."""
    def __init__(self, x, y_ovelha):
        self.x = x
        self.frames = frames_passaro
        self.frame_atual = 0
        self.temporizador_animacao = 0
        self.tamanho = TAMANHO_PASSARO
        self.eh_rapido = random.random() < 0.3
        
        # Ajuste de velocidade conforme solicitado
        if self.eh_rapido:
            self.velocidade = VELOCIDADE_NORMAL_MULTIPLIER * VELOCIDADE_RAPIDO_MULTIPLIER
        else:
            self.velocidade = VELOCIDADE_NORMAL_MULTIPLIER
        
        self.y = Y_CHAO - self.tamanho - 50 if random.random() < 0.5 else y_ovelha - ALTURA_TELA * DESLOCAMENTO_ALTURA_PASSARO
        self.y = max(ALTURA_TELA * 0.1, min(Y_CHAO - self.tamanho, self.y))
        
        # Adicionado: Cria um retângulo para colisão, mesmo que o desenho falhe
        self.retangulo = pygame.Rect(self.x, self.y, self.tamanho, self.tamanho)

    def atualizar(self, velocidade_jogo, x_ovelha, y_ovelha, dt):
        """Atualiza a posição e animação do pássaro."""
        self.temporizador_animacao += dt
        if self.temporizador_animacao >= ATRASO_ANIMACAO_PASSARO:
            self.temporizador_animacao = 0
            self.frame_atual = (self.frame_atual + 1) % len(self.frames)
        
        self.x -= velocidade_jogo * self.velocidade
        
        dx, dy = x_ovelha - self.x, y_ovelha - self.y
        dist = max(1, math.sqrt(dx*dx + dy*dy))
        self.x += dx/dist * FATOR_PERSEGUICAO
        self.y += dy/dist * FATOR_PERSEGUICAO
        self.y = max(ALTURA_TELA * 0.1, min(Y_CHAO - self.tamanho, self.y))
        
        self.retangulo.left = self.x
        self.retangulo.top = self.y
        
        return self.x < -self.tamanho
    
    def desenhar(self, superficie):
        """Desenha o pássaro na tela."""
        if self.frames:
            frame = self.frames[self.frame_atual % len(self.frames)]
            superficie.blit(frame, (self.x, self.y))
        else:
            pygame.draw.circle(superficie, PRETO, (int(self.x + self.tamanho/2), int(self.y + self.tamanho/2)), self.tamanho // 2)


class Pedra:
    """Representa a pedra, um obstáculo no chão."""
    def __init__(self, x):
        self.x = x
        self.imagem = random.choice([img_pedra1, img_pedra2])
        if not self.imagem or self.imagem.get_size() == (1,1):
            self.imagem = pygame.Surface((TAMANHO_PEDRA * 2, TAMANHO_PEDRA * 2))
            pygame.draw.rect(self.imagem, (100, 100, 100), (0, 0, TAMANHO_PEDRA * 2, TAMANHO_PEDRA * 2))
        
        self.retangulo = self.imagem.get_rect()
        self.retangulo.bottom = Y_CHAO
        self.retangulo.left = x
        self.tamanho = self.retangulo.width
    
    def atualizar(self, velocidade_jogo):
        """Atualiza a posição da pedra."""
        self.x -= velocidade_jogo
        self.retangulo.left = self.x
        return self.x < -self.tamanho
    
    def desenhar(self, superficie):
        """Desenha a pedra na tela."""
        superficie.blit(self.imagem, self.retangulo)

def mostrar_texto(superficie, texto, tamanho, cor, x, y, alinhamento="centro"):
    """Exibe um texto na tela com alinhamento flexível."""
    fonte = pygame.font.SysFont(None, tamanho)
    superficie_texto = fonte.render(texto, True, cor)
    retangulo_texto = superficie_texto.get_rect()
    if alinhamento == "centro":
        retangulo_texto.center = (x, y)
    elif alinhamento == "esquerda":
        retangulo_texto.midleft = (x, y)
    superficie.blit(superficie_texto, retangulo_texto)

def menu_inicial(recorde):
    """Tela de introdução do jogo."""
    pygame.mixer.stop()
    if sons['menu']:
        sons['menu'].play(-1)
    
    while True:
        tela.fill(AZUL_CEU)
        
        centro_x = LARGURA_TELA // 2
        y_titulo = ALTURA_TELA // 3
        y_iniciar = ALTURA_TELA // 2
        y_recorde = ALTURA_TELA // 1.5
        
        mostrar_texto(tela, "Sheep Runner", int(ALTURA_TELA * 0.08), PRETO, centro_x, y_titulo)
        mostrar_texto(tela, "Toque para Iniciar", int(ALTURA_TELA * 0.04), PRETO, centro_x, y_iniciar)
        mostrar_texto(tela, f"Recorde: {recorde:.2f}", int(ALTURA_TELA * 0.03), PRETO, centro_x, y_recorde)
        
        pygame.display.update()
        
        for evento in pygame.event.get():
            if evento.type == pygame.QUIT or (evento.type == pygame.KEYDOWN and evento.key == pygame.K_ESCAPE):
                pygame.quit()
                sys.exit()
            if evento.type == pygame.MOUSEBUTTONDOWN:
                if sons['menu']:
                    sons['menu'].stop()
                return

def tela_fim_de_jogo(pontuacao, recorde, novo_recorde):
    """Tela de Game Over."""
    pygame.mixer.stop()
    
    sobreposicao = pygame.Surface((LARGURA_TELA, ALTURA_TELA))
    sobreposicao.set_alpha(180)
    sobreposicao.fill(PRETO)
    tela.blit(sobreposicao, (0, 0))
    
    centro_x = LARGURA_TELA // 2
    
    mostrar_texto(tela, "Fim de Jogo", int(ALTURA_TELA * 0.06), BRANCO, centro_x, ALTURA_TELA//3)
    mostrar_texto(tela, f"Pontuação: {pontuacao:.2f}", int(ALTURA_TELA * 0.05), BRANCO, centro_x, ALTURA_TELA//2)
    
    y_mensagem = ALTURA_TELA//2 + int(ALTURA_TELA * 0.08)
    if novo_recorde:
        mostrar_texto(tela, "PARABÉNS! NOVO RECORDE!", int(ALTURA_TELA * 0.05), OURO, centro_x, y_mensagem)
        if sons['recorde']:
            sons['recorde'].play()
    else:
        mostrar_texto(tela, "Você não ultrapassou seu recorde", int(ALTURA_TELA * 0.035), BRANCO, centro_x, y_mensagem)
        mostrar_texto(tela, "Você consegue! Tente de novo!", int(ALTURA_TELA * 0.035), BRANCO, centro_x, y_mensagem + int(ALTURA_TELA * 0.05))
        if sons['falha']:
            sons['falha'].play()
    
    mostrar_texto(tela, "Toque para continuar", int(ALTURA_TELA * 0.03), BRANCO, centro_x, ALTURA_TELA * 0.7)
    
    pygame.display.update()
    
    esperando = True
    while esperando:
        for evento in pygame.event.get():
            if evento.type == pygame.QUIT or (evento.type == pygame.KEYDOWN and evento.key == pygame.K_ESCAPE):
                pygame.quit()
                sys.exit()
            if evento.type == pygame.MOUSEBUTTONDOWN:
                esperando = False
        
        pygame.time.Clock().tick(60)

def main():
    """Função principal do jogo."""
    relogio = pygame.time.Clock()
    recorde = 0
    
    while True:
        menu_inicial(recorde)
        
        ovelha = Ovelha()
        chao = Chao()
        nuvens = [Nuvem() for _ in range(3)]
        obstaculos = []
        pontuacao = 0
        velocidade_jogo = VELOCIDADE_JOGO
        ultima_geracao_obstaculo = pygame.time.get_ticks()
        temporizador_piscar = 0
        novo_recorde = False
        
        def gerar_grupo_obstaculo():
            """Gera um grupo de obstáculos aleatório."""
            grupo = []
            combo = random.choice(["duas_pedras", "dois_passaros", "misto"])
            
            x = LARGURA_TELA
            if combo == "duas_pedras":
                grupo.append(Pedra(x))
                grupo.append(Pedra(x + DISTANCIA_INTERNA_GRUPO))
            elif combo == "dois_passaros":
                grupo.append(Passaro(x, ovelha.y))
                grupo.append(Passaro(x + DISTANCIA_INTERNA_GRUPO, ovelha.y))
            else:
                tipos = random.choice([['pedra', 'passaro'], ['passaro', 'pedra']])
                for i, tipo_obs in enumerate(tipos):
                    if tipo_obs == 'passaro':
                        grupo.append(Passaro(x + i * DISTANCIA_INTERNA_GRUPO, ovelha.y))
                    else:
                        grupo.append(Pedra(x + i * DISTANCIA_INTERNA_GRUPO))
            return grupo

        rodando = True
        while rodando:
            dt = relogio.tick(60)
            
            for evento in pygame.event.get():
                if evento.type == pygame.QUIT or (evento.type == pygame.KEYDOWN and evento.key == pygame.K_ESCAPE):
                    pygame.quit()
                    sys.exit()
                if evento.type == pygame.MOUSEBUTTONDOWN and not ovelha.esta_morta:
                    ovelha.pular()
            
            if ovelha.atualizar():
                rodando = False
                
            if not ovelha.esta_morta:
                pontuacao += 0.1
                velocidade_jogo += INCREMENTO_VELOCIDADE
                
                if pontuacao > recorde and not novo_recorde:
                    novo_recorde = True
                    temporizador_piscar = DURACAO_PISCAR
                    if sons['pontuacao']:
                        sons['pontuacao'].play()
                
                tempo_atual = pygame.time.get_ticks()
                if not obstaculos and tempo_atual - ultima_geracao_obstaculo > max(500, 1500 - min(1000, pontuacao)):
                    obstaculos.extend(gerar_grupo_obstaculo())
                    ultima_geracao_obstaculo = tempo_atual
            
            for nuvem in nuvens:
                nuvem.atualizar(velocidade_jogo)
            
            for obstaculo in obstaculos[:]:
                if (isinstance(obstaculo, Passaro) and obstaculo.atualizar(velocidade_jogo, ovelha.x, ovelha.y, dt)) or \
                   (isinstance(obstaculo, Pedra) and obstaculo.atualizar(velocidade_jogo)):
                    obstaculos.remove(obstaculo)
            
            # Corrigido: Verificação de colisão mais segura
            if not ovelha.esta_morta and ovelha.animacao_atual:
                frame_idx = min(ovelha.frame_atual, len(ovelha.animacao_atual) - 1)
                frame = ovelha.animacao_atual[frame_idx]
                retangulo_ovelha = frame.get_rect(midbottom=(ovelha.x, ovelha.y + ovelha.raio))
                
                for obstaculo in obstaculos:
                    # Melhoramos a detecção de colisão para passáros, usando o retângulo criado
                    retangulo_obstaculo = obstaculo.retangulo if isinstance(obstaculo, Pedra) else obstaculo.retangulo
                    if retangulo_ovelha.colliderect(retangulo_obstaculo):
                        ovelha.iniciar_morte()
                        break
            
            tela.fill(AZUL_CEU)
            
            for nuvem in nuvens:
                nuvem.desenhar(tela)
            
            chao.atualizar(velocidade_jogo)
            chao.desenhar(tela)
            
            texto_pontuacao = f"Pontuação: {pontuacao:.2f}"
            tamanho_pontuacao = int(ALTURA_TELA * 0.04)
            x_pontuacao = 20
            y_pontuacao = ALTURA_TELA // 2
            
            if temporizador_piscar > 0:
                temporizador_piscar -= 1
                if (temporizador_piscar // 10) % 2 == 0:
                    mostrar_texto(tela, texto_pontuacao, tamanho_pontuacao, OURO, x_pontuacao, y_pontuacao, "esquerda")
                else:
                    mostrar_texto(tela, texto_pontuacao, tamanho_pontuacao, BRANCO, x_pontuacao, y_pontuacao, "esquerda")
            else:
                mostrar_texto(tela, texto_pontuacao, tamanho_pontuacao, BRANCO, x_pontuacao, y_pontuacao, "esquerda")
            
            ovelha.desenhar(tela)
            for obstaculo in obstaculos:
                obstaculo.desenhar(tela)
            
            pygame.display.update()
        
        if pontuacao > recorde:
            recorde = pontuacao
        
        tela_fim_de_jogo(pontuacao, recorde, novo_recorde)

if __name__ == "__main__":
    main()
