# Como desenhar o rosto do robô (sem usar imagem/emoji)

Este guia mostra como trocar o modo atual (emoji/imagem via `SetEmotion`) por desenho vetorial com LVGL.

## 1) Onde o rosto é controlado hoje

No fluxo atual, o rosto é trocado por nome de emoção:

- `EmoteDisplay::SetEmotion()` chama `emote_set_anim_emoji(...)` e usa assets de emoji.
- `LvglDisplay::SetPowerSaveMode()` também chama `SetEmotion("sleepy")` e `SetEmotion("neutral")`.

Ou seja: para **não depender de imagem**, você deve desenhar os elementos do rosto diretamente na tela (olhos, boca, sobrancelhas) com objetos LVGL.

## 2) Estratégia recomendada

A abordagem mais simples e legível é:

1. Criar um display próprio (ex.: `RobotFaceDisplay`) baseado em `LvglDisplay`.
2. Criar objetos LVGL fixos para o rosto:
   - `left_eye_` e `right_eye_` (`lv_obj_create`, retângulos arredondados).
   - `mouth_` (`lv_line` ou `lv_arc`).
3. Em `SetEmotion(...)`, **mudar apenas estilos/posições** desses objetos (altura do olho, raio da boca, cor etc.).
4. Manter a mesma interface pública (`SetEmotion`, `SetStatus`, `SetChatMessage`) para não quebrar o restante do app.

## 3) Exemplo de implementação (mínima)

> Exemplo didático para iniciar rápido. Ajuste tamanhos conforme sua tela.

```cpp
class RobotFaceDisplay : public LvglDisplay {
public:
    RobotFaceDisplay(...) : LvglDisplay(...) {
        CreateFaceWidgets();
        SetEmotion("neutral");
    }

    void SetEmotion(const char* emotion) override {
        DisplayLockGuard lock(this);
        if (emotion == nullptr) return;

        if (strcmp(emotion, "neutral") == 0) {
            SetEyes(30, 30, 12);      // largura, altura, raio
            SetMouthSmile(0);         // 0 = reta
        } else if (strcmp(emotion, "happy") == 0) {
            SetEyes(30, 24, 12);
            SetMouthSmile(20);        // sorriso
        } else if (strcmp(emotion, "sleepy") == 0) {
            SetEyes(34, 6, 3);        // olhos quase fechados
            SetMouthSmile(-8);        // boca levemente triste
        }
    }

private:
    lv_obj_t* left_eye_ = nullptr;
    lv_obj_t* right_eye_ = nullptr;
    lv_obj_t* mouth_ = nullptr;

    void CreateFaceWidgets() {
        left_eye_ = lv_obj_create(container_);
        right_eye_ = lv_obj_create(container_);
        mouth_ = lv_line_create(container_);

        // Remover estilos padrão para virar "shape"
        lv_obj_remove_style_all(left_eye_);
        lv_obj_remove_style_all(right_eye_);

        lv_obj_set_style_bg_color(left_eye_, lv_color_white(), 0);
        lv_obj_set_style_bg_opa(left_eye_, LV_OPA_COVER, 0);
        lv_obj_set_style_bg_color(right_eye_, lv_color_white(), 0);
        lv_obj_set_style_bg_opa(right_eye_, LV_OPA_COVER, 0);

        lv_obj_align(left_eye_, LV_ALIGN_CENTER, -35, -18);
        lv_obj_align(right_eye_, LV_ALIGN_CENTER, 35, -18);
        lv_obj_align(mouth_, LV_ALIGN_CENTER, 0, 22);
    }

    void SetEyes(int w, int h, int radius) {
        lv_obj_set_size(left_eye_, w, h);
        lv_obj_set_size(right_eye_, w, h);
        lv_obj_set_style_radius(left_eye_, radius, 0);
        lv_obj_set_style_radius(right_eye_, radius, 0);
    }

    void SetMouthSmile(int curve) {
        static lv_point_precise_t pts[3];
        pts[0] = { -24, 0 };
        pts[1] = { 0, curve };
        pts[2] = { 24, 0 };

        lv_line_set_points(mouth_, pts, 3);
        lv_obj_set_style_line_width(mouth_, 4, 0);
        lv_obj_set_style_line_color(mouth_, lv_color_white(), 0);
    }
};
```

## 4) Onde ligar no projeto

1. Substitua a criação do display da sua placa para usar `RobotFaceDisplay` no lugar do display atual (`SpiLcdDisplay`, `LcdDisplay` ou `EmoteDisplay`, dependendo da board).
2. Mantenha chamadas existentes de `SetEmotion("...")`: elas continuarão funcionando, mas agora alterando desenho, não emoji.
3. Se não quiser assets de emoji, remova a dependência de `DEFAULT_EMOJI_COLLECTION` na configuração da board.

## 5) Boas práticas

- **Separar criação e animação**:
  - `CreateFaceWidgets()` só cria objetos.
  - `SetEmotion()` só muda estado visual.
- **Evitar recriar objetos em loop**: crie uma vez e só atualize estilo/posição.
- **Padronizar emoções** com nomes fixos (`neutral`, `happy`, `sleepy`, `angry`) para simplificar integração com o restante do firmware.
- **Comentar regras visuais** importantes (ex.: “sleepy = altura do olho menor”).

## 6) Plano de migração em pequenas tarefas

1. Criar classe `RobotFaceDisplay` e desenhar rosto `neutral`.
2. Implementar `happy` e `sleepy`.
3. Trocar apenas uma board para validar.
4. Ajustar posições/tamanhos para a resolução alvo.
5. Expandir para mais emoções quando estável.

Com isso, você deixa o rosto 100% desenhado por código, sem depender de PNG/GIF para expressões.
