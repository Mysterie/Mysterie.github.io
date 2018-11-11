---
layout: post
title:  "A la quête du frame buffer"
date:   2009-07-22 15:08:00 +0100
categories: [security]
---
C'est l'été, dehors il fait beau, les oiseaux chantent. Autant de bonnes raisons pour rester chez moi et coder.

Toujours à la recherche de mon frame buffer, il existe quelques méthodes pour récupérer l'adresse physique de celui-ci, mais rien pour l'adresse virtuelle. J'ai eu l'idée d'envoyer des `IOCTL` au driver miniport de ma carte graph' pour qu'il me renvoie les informations que je recherche. Pour ce faire j'utilise la fonction [DeviceioControl()](https://docs.microsoft.com/fr-fr/windows/desktop/api/ioapiset/nf-ioapiset-deviceiocontrol) qui nécessite un handle. Ce handle, on le récupère avec [CreateFile("\\\\.\\DISPLAY1, ...);](https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-createfilea) sauf qu'en retour, on a le droit à un beau "Access denied". ouch...

`DISPLAY1` est un lien symbolique vers `\Device\Video0`, chaque device est lié à un driver, sous VMware on a:

```
kd> !devobj Video0
Device object (8163e870) is for:
 Video0 \Driver\vmx_svga DriverObject 8164f880

kd> !drvobj vmx_svga 3
Driver object (8164f880) is for:
 \Driver\vmx_svga
Driver Extension List: (id , addr)
(8164f880 816e47f8)  
Device Object list:
8163e870  

DriverEntry:   f9eee240	vmx_svga
DriverStartIo: 00000000
DriverUnload:  f97aa7e6	VIDEOPRT!VpDriverUnload
AddDevice:     f97ac418	VIDEOPRT!VpAddDevice

Dispatch routines:
[00] IRP_MJ_CREATE                      f97ab65c	VIDEOPRT!pVideoPortDispatch
[01] IRP_MJ_CREATE_NAMED_PIPE           805025e4	nt!IopInvalidDeviceRequest
[02] IRP_MJ_CLOSE                       f97ab65c	VIDEOPRT!pVideoPortDispatch
[03] IRP_MJ_READ                        805025e4	nt!IopInvalidDeviceRequest
[04] IRP_MJ_WRITE                       805025e4	nt!IopInvalidDeviceRequest
[05] IRP_MJ_QUERY_INFORMATION           805025e4	nt!IopInvalidDeviceRequest
[06] IRP_MJ_SET_INFORMATION             805025e4	nt!IopInvalidDeviceRequest
[07] IRP_MJ_QUERY_EA                    805025e4	nt!IopInvalidDeviceRequest
[08] IRP_MJ_SET_EA                      805025e4	nt!IopInvalidDeviceRequest
[09] IRP_MJ_FLUSH_BUFFERS               805025e4	nt!IopInvalidDeviceRequest
[0a] IRP_MJ_QUERY_VOLUME_INFORMATION    805025e4	nt!IopInvalidDeviceRequest
[0b] IRP_MJ_SET_VOLUME_INFORMATION      805025e4	nt!IopInvalidDeviceRequest
[0c] IRP_MJ_DIRECTORY_CONTROL           805025e4	nt!IopInvalidDeviceRequest
[0d] IRP_MJ_FILE_SYSTEM_CONTROL         805025e4	nt!IopInvalidDeviceRequest
[0e] IRP_MJ_DEVICE_CONTROL              f97ab65c	VIDEOPRT!pVideoPortDispatch
[0f] IRP_MJ_INTERNAL_DEVICE_CONTROL     805025e4	nt!IopInvalidDeviceRequest
[10] IRP_MJ_SHUTDOWN                    f97ab65c	VIDEOPRT!pVideoPortDispatch
[11] IRP_MJ_LOCK_CONTROL                805025e4	nt!IopInvalidDeviceRequest
[12] IRP_MJ_CLEANUP                     805025e4	nt!IopInvalidDeviceRequest
[13] IRP_MJ_CREATE_MAILSLOT             805025e4	nt!IopInvalidDeviceRequest
[14] IRP_MJ_QUERY_SECURITY              805025e4	nt!IopInvalidDeviceRequest
[15] IRP_MJ_SET_SECURITY                805025e4	nt!IopInvalidDeviceRequest
[16] IRP_MJ_POWER                       f97a7d80	VIDEOPRT!pVideoPortPowerDispatch
[17] IRP_MJ_SYSTEM_CONTROL              f979f748	VIDEOPRT!pVideoPortSystemControl
[18] IRP_MJ_DEVICE_CHANGE               805025e4	nt!IopInvalidDeviceRequest
[19] IRP_MJ_QUERY_QUOTA                 805025e4	nt!IopInvalidDeviceRequest
[1a] IRP_MJ_SET_QUOTA                   805025e4	nt!IopInvalidDeviceRequest
[1b] IRP_MJ_PNP                         f97a7150	VIDEOPRT!pVideoPortPnpDispatch
```

Hop un petit schéma pour les autistes:

```
   __________________                         _____________
  |                  |---------------------->| I/O Manager |____
  | GDI (win32k.sys) |  GreDeviceIoControl +>|_(Video0)____|    |
+>|__________________| |                   |                    |
| Eng call             |                   |  ________________  |
|                      |                   |_| Video Port     |<+
|  _________Ddi call_  |                   +>|_(Videoprt.sys)_| |
|_|                  |<+                   |  ________________  |
  | Display driver   |                     |_| Video Miniport |<+
+>|_(vmx_fb.dll)_____|                       |_(vmx_svga.sys)_|<--+
|          ___________________________________________            |
|_________|             La carte Graph \o/            |___________|
```

Grâce à un petit script made in Invanlef0u on va pouvoir retrouver les handles qui nous intéressent. J'ai aussi [codé un driver](http://mysterie.fr/prog/blog/dispHandle.htm) qui dump l'object table du process System.

```
kd>bp nt!zwopenfile ".printf \"ZwOpenFile \"; !ustr poi(poi(@esp+4+2*4)+2*4);"

// Ca break sur \Device\Video0
kd> !handle 80000478
processor number 0, process 815c3020
PROCESS 815c3020  SessionId: 0  Cid: 025c    Peb: 7ffde000  ParentCid: 0168
    DirBase: 07136000  ObjectTable: e148c9a0  HandleCount:  25.
    Image: csrss.exe

Kernel Handle table at e1539000 with 58 Entries in use
80000478: Object: 816e48e8  GrantedAccess: 00000000 Entry: e10028f0
Object: 816e48e8  Type: (817b8730) File
    ObjectHeader: 816e48d0 (old version)
        HandleCount: 1  PointerCount: 2
```

L'handle est ouvert avec `DesiredAccess = FILE_SUPERSEDED` et `ShareAccess = 0`. Aïe pas moyen d'interagir avec. D'ailleurs, quand le device est chargé, on se rend compte que le handle a été refermé. Il y a juste un pointeur sur l'objet `video0`. Donc, on ne cherche pas dans la bonne direction... :(

Il existe une fonction noyau équivalente à [DeviceioControl](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/ntifs/nf-ntifs-ntdeviceiocontrolfile) qui est [IoBuildDeviceIoControlRequest](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/wdm/nf-wdm-iobuilddeviceiocontrolrequest) (merci [Overclok](http://0vercl0k.tuxfamily.org/bl0g/) pour l'indication). Elle crée des irp de type `IRP_MJ_INTERNAL_DEVICE_CONTROL` et `IRP_MJ_DEVICE_CONTROL`. C'est le deuxième type qui nous intéresse, le driver vidéo n'implante pas la première. Et pour cette fonction, nul besoin d'un handle, un pointeur vers l'objet video0 suffit.

Je crée un autre bp sur la fonction [IoGetDeviceObjectPointer()](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/wdm/nf-wdm-iogetdeviceobjectpointer) pour trouver ce pointeur au boot.

```
kd>bp nt!IoGetDeviceObjectPointer "!ustr poi(@esp+4); .printf \"\n\tFile:\"; dd poi(@esp+4+4*2) L1; .printf \"\n\tDevice:\"; dd poi(@esp+4+4*3) L1;"
kd>g
String(28,30) at f9c7ac44: \Device\Video0
 	File:f9c7ac68  f9c7ad18
 	Device:f9c7ac84  00000000
nt!IoGetDeviceObjectPointer:
805904b9 8bff            mov     edi,edi
kd>ba w 1 0xf9c7ac84

//Puis ca break le pointeur vers video0 est initialisé.
kd> dt nt!_DEVICE_OBJECT 0x816cac30
   +0x000 Type             : 3
   +0x002 Size             : 0x3a0
   +0x004 ReferenceCount   : 1
   +0x008 DriverObject     : 0x816ce488 _DRIVER_OBJECT
   +0x00c NextDevice       : (null)
   +0x010 AttachedDevice   : (null)
   +0x014 CurrentIrp       : (null)
   +0x018 Timer            : (null)
   +0x01c Flags            : 0x284c
   +0x020 Characteristics  : 0x100
   +0x024 Vpb              : (null)
   +0x028 DeviceExtension  : 0x816cace8
   +0x02c DeviceType       : 0x23
   +0x030 StackSize        : 2 ''
   +0x034 Queue            : __unnamed
   +0x05c AlignmentRequirement : 0
   +0x060 DeviceQueue      : _KDEVICE_QUEUE
   +0x074 Dpc              : _KDPC
   +0x094 ActiveThreadCount : 0
   +0x098 SecurityDescriptor : 0xe14b3680
   +0x09c DeviceLock       : _KEVENT
   +0x0ac SectorSize       : 0
   +0x0ae Spare1           : 0
   +0x0b0 DeviceObjectExtension : 0x816cafd0 _DEVOBJ_EXTENSION
   +0x0b4 Reserved         : (null)
```

Ensuite il suffit d'envoyer un IoControlCode de type [IOCTL_VIDEO_MAP_VIDEO_MEMORY](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/ntddvdeo/ni-ntddvdeo-ioctl_video_reset_device) avec les structures [VIDEO_MEMORY](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/ntddvdeo/ns-ntddvdeo-_video_memory) et [VIDEO_MEMORY_INFORMATION](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/ntddvdeo/ns-ntddvdeo-_video_memory_information) à zero. Et on obtient les adresses virtuelles de la mémoire vidéo et du frame buffer, dans mon cas:

```
Video Memory:
0xf86d8000 size: 16777216
Frame buffer:
0xf86d8000 size: 1920000

kd> dd 0xf86d8000
f86d8000  00edf6f5 00ecf5f4 00edf2f5 00ecf1f4
f86d8010  00ecf1f5 00ecf1f5 00edf0f7 00edf0f7
f86d8020  00f1f2f7 00f2f3f7 00f4f5f9 00f6f8f7
f86d8030  00f7f7f5 00f7f8f3 00f5f6f0 00f5f6f0
```

Voilà les premiers pixels de l'écran. Mais ce qu'il y a de plus intéressant c'est la taille du frame buffer par rapport à la taille de la mémoire vidéo. `16777216-1920000 = 14857216` octet. Donc **14,857Mo** dans le kernel space non utilisé où on peut y cacher plein de choses. Étonnant, non?

J'utilise `ObReferenceObjectByName()` fonction non documentée, avec le nom du driver miniport "vmx_svga", pour récupérer le pointeur sur le device video0. D'ailleurs, grâce à cette fonction, on pourrait lister tous les devices chargés, et leur envoyer des IOCTL, moi je me suis limité à la vidéo.

Si vous voulez tester le code de votre côté, pour retrouver le nom du driver vidéo il suffit de chercher dans les registres.
`HKEY_LOCAL_MACHINE\HARDWARE\DEVICEMAP\VIDEO clef \Device\Video0` contient un identifiant chez moi `E27CB637-47DA-4B2B-8B0D-50E8CD7B9FAC`. On s'en sert ensuite
`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Video\{E27CB637-47DA-4B2B-8B0D-50E8CD7B9FAC}\Video` et la clef `Service` contient le nom de notre driver miniport. Donc dans mon code il faudra remplacer le nom du driver "vmx_svga" par le vôtre.  
Oui j'ai bâclé la fin. J'éditerai surement le post pour automatiser la recherche, ou pas.

Code: framebuf.sys
