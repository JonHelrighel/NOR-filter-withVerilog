FIR Notch Filter em Verilog — DE10-Lite Filtro FIR notch (rejeita-faixa) que fiz pra disciplina de Linguagens de Descrição de Hardware, pra rodar no kit DE10-Lite (FPGA Intel MAX10), usando Quartus Prime 18.1.

A proposta era filtrar o ruído de rede elétrica (60 Hz) de um sinal de áudio de 48 kHz, dentro de uma cadeia maior: entrada analógica → ADC → este filtro FIR → DAC via SPI → saída analógica. Esse repositório aqui é só a parte que eu implementei: o bloco do filtro em si e o testbench dele. O resto da cadeia (wrapper do ADC, driver SPI do DAC) não é meu e não faz parte deste repo.

Como o filtro funciona É um FIR de 16 taps, implementado como um MAC sequencial (multiplica-acumula um par por ciclo de clock, em vez de 16 multiplicadores em paralelo) — faz sentido porque a 48 kHz sobra bastante tempo de clock (a placa roda a 50 MHz) pra processar tudo sequencialmente sem precisar gastar hardware com multiplicadores paralelos.

Resumindo o fir_notch.v:

Chega uma amostra nova (nova_amostra em nível alto) → desloca o shift register de 16 posições (x0...x15), a mais nova entra em x0. Ao mesmo tempo dispara o MAC: percorre par = 0..15, multiplicando cada x[i] pelo coeficiente H[i] correspondente e acumulando em soma. Depois do 16º ciclo (par == 15), o resultado sai em saida (pega os bits [27:16] do acumulador, pra descartar o crescimento de bits da multiplicação) e saida_valida sobe por 1 ciclo. Os coeficientes ficam em coeff_pkg.vh, definidos como `define (H0 a H15).

Estado atual / o que ainda falta Sendo honesto: os coeficientes que estão em coeff_pkg.vh agora são valores de teste (uma rampa simétrica 100→450→100), não os coeficientes reais calculados pra atenuar 60 Hz especificamente. Isso ainda precisa ser gerado com scipy.signal.firwin (ou uma FDA Tool) e substituído no coeff_pkg.vh antes de considerar o filtro "pronto" de verdade — a estrutura do hardware (shift register + MAC sequencial) já está funcionando, só falta o cálculo dos coeficientes certos.

Também não implementei a otimização de simetria de fase linear (somar x[k] + x[15-k] antes de multiplicar, reduzindo de 16 pra 8 multiplicações) nem a versão shift-add em vez do operador *. Fica como próximo passo se eu quiser deixar mais enxuto.

Testbench tb_fir.v aplica um impulso (zeros, depois uma amostra em 2047, depois mais zeros) e imprime tempo, entrada, saída toda vez que saida_valida sobe — dá pra ver a resposta ao impulso do filtro direto no console do simulador.

Pra rodar no ModelSim (via linha de comando):

vlib work vlog rtl/fir_notch.v tb/tb_fir.v vsim -novopt tb_fir add wave * run -all Estrutura . ├── rtl/ │ ├── fir_notch.v → o filtro em si │ └── coeff_pkg.vh → coeficientes (ainda valores de teste, ver acima) ├── tb/ │ └── tb_fir.v → testbench (resposta ao impulso) └── quartus/ ├── fir_notch_de10lite.qpf └── fir_notch_de10lite.qsf → projeto Quartus (MAX10 10M50DAF484C6GES, top-level = fir_notch)
