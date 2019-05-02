# redes1
Para compartir código de simuladores en Telecomunicaciones 1.
#define MAX_SEQ 7 /* debe ser 2^n – 1 */
#define NR_BUFS ((MAX_SEQ 1 1)/2)
typedef enum {frame_arrival, cksum_err, timeout, network_layer_ready, ack_timeout} event_type;
#include “protocol.h”
boolean no_nak 5 true; /* aún no se ha enviado un nak */
seq_nr oldest_frame 5 MAX_SEQ 1 1; /* el valor inicial es sólo para el simulador */
static boolean between(seq_nr a, seq_nr b, seq_nr c)
{
/* Parecido a lo que ocurre en el protocolo 5, pero más corto y confuso. */
return ((a <5 b) && (b < c)) || ((c < a) && (a <5 b)) || ((b < c) && (c < a));
}
static void send_frame(frame_kind fk, seq_nr frame_nr, seq_nr frame_expected, packet buffer[])
{
/* Construye y envía una trama de datos, ack o nak. */
frame s; /* variable de trabajo */
s.kind 5 fk; /* kind 55 datos, ack o nak */
if (fk 55 data) s.info 5 buffer[frame_nr % NR_BUFS];
s.seq 5 frame_nr; /* sólo tiene importancia para las tramas de datos */
s.ack 5 (frame_expected 1 MAX_SEQ) % (MAX_SEQ 1 1);
if (fk 55 nak) no_nak 5 false; /* un nak por trama, por favor */
to_physical_layer(&s); /* transmite la trama */
if (fk 55 data) start_timer(frame_nr % NR_BUFS);
stop_ack_timer(); /* no se necesita para tramas ack independientes */
}



void protocol6(void)
{
seq_nr ack_expected; /* límite inferior de la ventana del emisor */
seq_nr next_frame_to_send; /* límite superior de la ventana del emisor 1 1 */
seq_nr frame_expected; /* límite inferior de la ventana del receptor */
seq_nr too_far; /* límite superior de la ventana del receptor 1 1 */
int i; /* índice en el grupo de búferes */
frame r; /* variable de trabajo*/
packet out_buf[NR_BUFS]; /* búferes para el flujo de salida */
packet in_buf[NR_BUFS]; /* búferes para el flujo de entrada */
boolean arrived[NR_BUFS]; /* mapa de bits de entrada */
seq_nr nbuffered; /* cuántos búferes de salida se utilizan actualmente */
event_type event;
enable_network_layer(); /* inicializar */
ack_expected 5 0; /* siguiente ack esperada en el flujo de entrada */
next_frame_to_send 5 0; /* número de la siguiente trama de salida */
frame_expected 5 0;
too_far 5 NR_BUFS;
nbuffered 5 0; /* al principio no hay paquetes en el búfer */
for (i 5 0; i < NR_BUFS; i11) arrived[i] 5 false;
while (true) {
wait_for_event(&event); /* cinco posibilidades: vea event_type al principio */
switch(event) {
 case network_layer_ready: /* acepta, guarda y transmite una trama nueva */ 
 
 nbuffered 5 nbuffered 1 1; /* expande la ventana */
 from_network_layer(&out_buf[next_frame_to_send % NR_BUFS]); /* obtiene un paquete nuevo */
 send_frame(data, next_frame_to_send, frame_expected, out_buf); /* transmite la trama */
 inc(next_frame_to_send); /* avanza el límite superior de la ventana */
 break;
 case frame_arrival: /* ha llegado una trama de datos o de control */
 from_physical_layer(&r); /* obtiene una trama entrante de la capa física */
 if (r.kind 55 data) {
 /* Ha llegado una trama no dañada. */
 if ((r.seq !5 frame_expected) && no_nak)
 send_frame(nak, 0, frame_expected, out_buf); else start_ack_timer();
 if (between(frame_expected, r.seq, too_far) && (arrived[r.seq%NR_BUFS]55 false)) {
 /* Las tramas se podrían aceptar en cualquier orden. */
 arrived[r.seq % NR_BUFS] 5 true; /* marca el búfer como lleno */
 in_buf[r.seq % NR_BUFS] 5 r.info; /* inserta datos en el búfer */
 while (arrived[frame_expected % NR_BUFS]) {
 /* Pasa tramas y avanza la ventana. */
 to_network_layer(&in_buf[frame_expected % NR_BUFS]);
 no_nak 5 true;
 arrived[frame_expected % NR_BUFS] 5 false;
 inc(frame_expected); /* avanza el límite inferior de la ventana del receptor */
 inc(too_far); /* avanza el límite superior de la ventana del receptor */
 start_ack_timer(); /* para saber si es necesaria una ack independiente */
 }
 }
 }
 if((r.kind55nak) && between(ack_expected,(r.ack11)%(MAX_SEQ11), next_frame_to_send))
 send_frame(data, (r.ack11) % (MAX_SEQ 1 1), frame_expected, out_buf);
 while (between(ack_expected, r.ack, next_frame_to_send)) {
 nbuffered 5 nbuffered - 1; /* maneja la ack superpuesta */
 stop_timer(ack_expected % NR_BUFS); /* la trama llega intacta */
 inc(ack_expected); /* avanza el límite inferior de la ventana del emisor */
 }
 break;
 case cksum_err:
 if (no_nak) send_frame(nak, 0, frame_expected, out_buf); /* trama dañada */
 break;
 case timeout:
 send_frame(data, oldest_frame, frame_expected, out_buf); /* expiró el temporizador */
 break;
 case ack_timeout:
 send_frame(ack,0,frame_expected, out_buf); /* expiró el temporizador de ack; envía ack */
}
if (nbuffered < NR_BUFS) enable_network_layer(); else disable_network_layer();
}
}






El protocolo 4 (ventana deslizante) es bidireccional. */
#define MAX_SEQ 1 /* debe ser 1 para el protocolo 4 */
typedef enum {frame_arrival, cksum_err, timeout} event_type;
#include “protocol.h”
void protocol4 (void)
{
seq_nr next_frame_to_send; /* sólo 0 o 1 */
seq_nr frame_expected; /* sólo 0 o 1 */
frame r, s; /* variables de trabajo */
packet buffer; /* paquete actual que se envía */
event_type event;
next_frame_to_send 5 0; /* siguiente trama del flujo de salida */
frame_expected 5 0; /* próxima trama esperada */
from_network_layer(&buffer); /* obtiene un paquete de la capa de red */
s.info 5 buffer; /* se prepara para enviar la trama inicial */
s.seq 5 next_frame_to_send; /* inserta el número de secuencia en la trama */
s.ack 5 1 – frame_expected; /* confirmación de recepción superpuesta */
to_physical_layer(&s); /* transmite la trama */
start_timer(s.seq); /* inicia el temporizador */
while (true){
 wait_for_event(&event); /* frame_arrival, cksum_err o timeout */
 if (event 55 frame_arrival){ /* ha llegado una trama sin daño. */
 from_physical_layer(&r); /* la obtiene */
 if(r.seq 55 frame_expected) { /* maneja flujo de tramas de entrada. */
 to_network_layer(&r.info); /* pasa el paquete a la capa de red */
 inc(frame_expected); /* invierte el siguiente número de secuencia esperado */
 }
 if(r.ack 55 next_frame_to_send){ /* maneja flujo de tramas de salida. */
 stop_timer(r.ack); /* desactiva el temporizador */
 from_network_layer(&buffer); /* obtiene un nuevo paquete de la capa de red */
 inc(next_frame_to_send); /* invierte el número de secuencia del emisor */
 }
 }
 s.info 5 buffer; /* construye trama de salida */
 s.seq 5 next_frame_to_send; /* le inserta el número de secuencia */
 s.ack 5 1 – frame_expected; /* número de secuencia de la última trama recibida */
 to_physical_layer(&s); /* transmite una trama */
 start_timer(s.seq); /* inicia el temporizador */
}
}

#define MAX_SEQ 7 /* debe ser 2^n – 1 */
#define NR_BUFS ((MAX_SEQ 1 1)/2)
typedef enum {frame_arrival, cksum_err, timeout, network_layer_ready, ack_timeout} event_type;
#include “protocol.h”
boolean no_nak 5 true; /* aún no se ha enviado un nak */
seq_nr oldest_frame 5 MAX_SEQ 1 1; /* el valor inicial es sólo para el simulador */
static boolean between(seq_nr a, seq_nr b, seq_nr c)
{
/* Parecido a lo que ocurre en el protocolo 5, pero más corto y confuso. */
return ((a <5 b) && (b < c)) || ((c < a) && (a <5 b)) || ((b < c) && (c < a));
}
static void send_frame(frame_kind fk, seq_nr frame_nr, seq_nr frame_expected, packet buffer[])
{
/* Construye y envía una trama de datos, ack o nak. */
frame s; /* variable de trabajo */
s.kind 5 fk; /* kind 55 datos, ack o nak */
if (fk 55 data) s.info 5 buffer[frame_nr % NR_BUFS];
s.seq 5 frame_nr; /* sólo tiene importancia para las tramas de datos */
s.ack 5 (frame_expected 1 MAX_SEQ) % (MAX_SEQ 1 1);
if (fk 55 nak) no_nak 5 false; /* un nak por trama, por favor */
to_physical_layer(&s); /* transmite la trama */
if (fk 55 data) start_timer(frame_nr % NR_BUFS);
stop_ack_timer(); /* no se necesita para tramas ack independientes */
}
void protocol6(void)
{
seq_nr ack_expected; /* límite inferior de la ventana del emisor */
seq_nr next_frame_to_send; /* límite superior de la ventana del emisor 1 1 */
seq_nr frame_expected; /* límite inferior de la ventana del receptor */
seq_nr too_far; /* límite superior de la ventana del receptor 1 1 */
int i; /* índice en el grupo de búferes */
frame r; /* variable de trabajo*/
packet out_buf[NR_BUFS]; /* búferes para el flujo de salida */
packet in_buf[NR_BUFS]; /* búferes para el flujo de entrada */
boolean arrived[NR_BUFS]; /* mapa de bits de entrada */
seq_nr nbuffered; /* cuántos búferes de salida se utilizan actualmente */
event_type event;
enable_network_layer(); /* inicializar */
ack_expected 5 0; /* siguiente ack esperada en el flujo de entrada */
next_frame_to_send 5 0; /* número de la siguiente trama de salida */
frame_expected 5 0;
too_far 5 NR_BUFS;
nbuffered 5 0; /* al principio no hay paquetes en el búfer */
for (i 5 0; i < NR_BUFS; i11) arrived[i] 5 false;
while (true) {
wait_for_event(&event); /* cinco posibilidades: vea event_type al principio */
switch(event) {
 case network_layer_ready: /* acepta, guarda y transmite una trama nueva */ 
 
 nbuffered 5 nbuffered 1 1; /* expande la ventana */
 from_network_layer(&out_buf[next_frame_to_send % NR_BUFS]); /* obtiene un paquete nuevo */
 send_frame(data, next_frame_to_send, frame_expected, out_buf); /* transmite la trama */
 inc(next_frame_to_send); /* avanza el límite superior de la ventana */
 break;
 case frame_arrival: /* ha llegado una trama de datos o de control */
 from_physical_layer(&r); /* obtiene una trama entrante de la capa física */
 if (r.kind 55 data) {
 /* Ha llegado una trama no dañada. */
 if ((r.seq !5 frame_expected) && no_nak)
 send_frame(nak, 0, frame_expected, out_buf); else start_ack_timer();
 if (between(frame_expected, r.seq, too_far) && (arrived[r.seq%NR_BUFS]55 false)) {
 /* Las tramas se podrían aceptar en cualquier orden. */
 arrived[r.seq % NR_BUFS] 5 true; /* marca el búfer como lleno */
 in_buf[r.seq % NR_BUFS] 5 r.info; /* inserta datos en el búfer */
 while (arrived[frame_expected % NR_BUFS]) {
 /* Pasa tramas y avanza la ventana. */
 to_network_layer(&in_buf[frame_expected % NR_BUFS]);
 no_nak 5 true;
 arrived[frame_expected % NR_BUFS] 5 false;
 inc(frame_expected); /* avanza el límite inferior de la ventana del receptor */
 inc(too_far); /* avanza el límite superior de la ventana del receptor */
 start_ack_timer(); /* para saber si es necesaria una ack independiente */
 }
 }
 }
 if((r.kind55nak) && between(ack_expected,(r.ack11)%(MAX_SEQ11), next_frame_to_send))
 send_frame(data, (r.ack11) % (MAX_SEQ 1 1), frame_expected, out_buf);
 while (between(ack_expected, r.ack, next_frame_to_send)) {
 nbuffered 5 nbuffered - 1; /* maneja la ack superpuesta */
 stop_timer(ack_expected % NR_BUFS); /* la trama llega intacta */
 inc(ack_expected); /* avanza el límite inferior de la ventana del emisor */
 }
 break;
 case cksum_err:
 if (no_nak) send_frame(nak, 0, frame_expected, out_buf); /* trama dañada */
 break;
 case timeout:
 send_frame(data, oldest_frame, frame_expected, out_buf); /* expiró el temporizador */
 break;
 case ack_timeout:
 send_frame(ack,0,frame_expected, out_buf); /* expiró el temporizador de ack; envía ack */
}
if (nbuffered < NR_BUFS) enable_network_layer(); else disable_network_layer();
}
}
/* El protocolo 4 (ventana deslizante) es bidireccional. */
#define MAX_SEQ 1 /* debe ser 1 para el protocolo 4 */
typedef enum {frame_arrival, cksum_err, timeout} event_type;
#include “protocol.h”
void protocol4 (void)
{
seq_nr next_frame_to_send; /* sólo 0 o 1 */
seq_nr frame_expected; /* sólo 0 o 1 */
frame r, s; /* variables de trabajo */
packet buffer; /* paquete actual que se envía */
event_type event;
next_frame_to_send 5 0; /* siguiente trama del flujo de salida */
frame_expected 5 0; /* próxima trama esperada */
from_network_layer(&buffer); /* obtiene un paquete de la capa de red */
s.info 5 buffer; /* se prepara para enviar la trama inicial */
s.seq 5 next_frame_to_send; /* inserta el número de secuencia en la trama */
s.ack 5 1 – frame_expected; /* confirmación de recepción superpuesta */
to_physical_layer(&s); /* transmite la trama */
start_timer(s.seq); /* inicia el temporizador */
while (true){
 wait_for_event(&event); /* frame_arrival, cksum_err o timeout */
 if (event 55 frame_arrival){ /* ha llegado una trama sin daño. */
 from_physical_layer(&r); /* la obtiene */
 if(r.seq 55 frame_expected) { /* maneja flujo de tramas de entrada. */
 to_network_layer(&r.info); /* pasa el paquete a la capa de red */
 inc(frame_expected); /* invierte el siguiente número de secuencia esperado */
 }
 if(r.ack 55 next_frame_to_send){ /* maneja flujo de tramas de salida. */
 stop_timer(r.ack); /* desactiva el temporizador */
 from_network_layer(&buffer); /* obtiene un nuevo paquete de la capa de red */
 inc(next_frame_to_send); /* invierte el número de secuencia del emisor */
 }
 }
 s.info 5 buffer; /* construye trama de salida */
 s.seq 5 next_frame_to_send; /* le inserta el número de secuencia */
 s.ack 5 1 – frame_expected; /* número de secuencia de la última trama recibida */
 to_physical_layer(&s); /* transmite una trama */
 start_timer(s.seq); /* inicia el temporizador */
}
}




