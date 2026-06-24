# Relatório — Controle Interativo de Cor e Intensidade (PWM + Terminal)

## Nome
Felipe Beserra de Oliveira

---

## Número USP
13683702

---

## Respostas, comentários e análises

### Descrição da Atividade

O experimento consiste em utilizar o módulo TPM2 da placa FRDM-KL25Z para gerar sinais PWM em dois canais — vermelho (PTB18) e verde (PTB19) — e combinar as cores para obter laranja. O usuário controla a intensidade via terminal serial (UART).

### Cor Laranja via PWM

Laranja é obtido pela mistura:

| Canal | Pino | Proporção de brilho |
|---|---|---|
| Vermelho | PTB18 (TPM2_CH0) | 100% |
| Verde | PTB19 (TPM2_CH1) | 65% |
| Azul | PTD1 | 0% (desligado) |

### Frequência do PWM

Com os seguintes parâmetros:
- Clock TPM: MCGFLLCLK ≈ 48 MHz (selecionado via `TPM_PLLFLL`)
- Prescaler: PS_128
- MOD: TPM_MODULE = 1000

A frequência resultante é:

```
f_PWM = f_CLK / (PS × MOD) = 48 MHz / (128 × 1000) ≈ 375 Hz
```

375 Hz está bem acima do limiar de percepção de *flicker* humano (~50–100 Hz), portanto o LED parece aceso continuamente.

### Lógica de Controle do Duty Cycle

O LED é *active low* (nível lógico 0 = ligado). No modo `TPM_PWM_H`, o pino fica HIGH durante `CnV/MOD` do período:

```
brilho_LED = 1 - CnV/MOD
```

Portanto, para um brilho desejado `B` (0–100%) e proporção de cor `C` (0–100%):

```c
CnV = TPM_MODULE × (100 - B × C / 100) / 100
```

Exemplos para `B = 100%`:
- Vermelho: `CnV = 1000 × (100 - 100) / 100 = 0` → LED 100% ligado
- Verde: `CnV = 1000 × (100 - 65) / 100 = 350` → LED 65% ligado

Para `B = 50%`:
- Vermelho: `CnV = 500`
- Verde: `CnV = 675`

### Interatividade via Terminal

O programa utiliza o subsistema de console do Zephyr (`console_getline()`) para ler uma linha digitada pelo usuário. O valor (0–100) é convertido para inteiro via `atoi()`, e os *duty cycles* de ambos os canais são recalculados mantendo a proporção laranja.

### Biblioteca utilizada

Foi utilizada a biblioteca customizada `pwm_z42` (acesso direto aos registradores do TPM), pois a configuração do *driver* Zephyr PWM para os canais TPM2_CH0 e TPM2_CH1 da FRDM-KL25Z requer *overlay* DTS adicional. A abordagem via registradores é equivalente funcionalmente e está alinhada com o conteúdo da disciplina sobre periféricos de hardware.

---

## Código (main.c)

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/sys/printk.h>
#include <zephyr/console/console.h>
#include <stdlib.h>
#include <pwm_z42.h>

#define TPM_MODULE      1000
#define ORANGE_RED_PCT   100
#define ORANGE_GREEN_PCT  65

static uint16_t brightness_to_cnv(int brightness_pct, int colour_pct)
{
    int effective = brightness_pct * colour_pct / 100;
    return (uint16_t)(TPM_MODULE * (100 - effective) / 100);
}

static void set_orange(int brightness_pct)
{
    if (brightness_pct < 0)   brightness_pct = 0;
    if (brightness_pct > 100) brightness_pct = 100;

    pwm_tpm_CnV(TPM2, 0, brightness_to_cnv(brightness_pct, ORANGE_RED_PCT));
    pwm_tpm_CnV(TPM2, 1, brightness_to_cnv(brightness_pct, ORANGE_GREEN_PCT));
}

int main(void)
{
    pwm_tpm_Init(TPM2, TPM_PLLFLL, TPM_MODULE, TPM_CLK, PS_128, EDGE_PWM);
    pwm_tpm_Ch_Init(TPM2, 0, TPM_PWM_H, GPIOB, 18);
    pwm_tpm_Ch_Init(TPM2, 1, TPM_PWM_H, GPIOB, 19);

    console_init();
    printk("=== Controle de Cor Laranja via PWM ===\n");
    set_orange(50);

    while (1) {
        printk("Digite a intensidade (0-100): ");
        char *line = console_getline();
        if (!line || line[0] == '\0') continue;

        int brightness = atoi(line);
        set_orange(brightness);

        printk("Intensidade: %d%% | CnV vermelho: %u | CnV verde: %u\n",
               brightness,
               brightness_to_cnv(brightness, ORANGE_RED_PCT),
               brightness_to_cnv(brightness, ORANGE_GREEN_PCT));
    }

    return 0;
}
```

---

## Repositório

```text
https://github.com/Beserrovsky/Atividade-2
```
