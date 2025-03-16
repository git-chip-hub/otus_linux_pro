#В плейбук добавлена готовая роль для работы с mdadm (mrlesmithjr.ansible-mdadm).
#Роль была переименована в ansible-mdadm. 
#Подробно ознакомился с плейбуками загруженной роли, разобрался как именно она работает. 
#Немного подкорректировал задачу удаления массива (включил игнорирование ошибок), т.к. после остановки массива блочное устройство удаляется (по крайней мере в Debian12) и из-за этого задача удаления падает с ошибкой.
#
#Для удобства экспериментов определил один элемент (state) переменной-списка (mdadm_array) роли (mrlesmithjr.ansible-mdadm) в дополнительную переменную (array_state). 
#Переменную array_state можно указать при запуске выполнения плейбука, если переменная не указана, то используется значение 'present'.

#создание массива
ansible-playbook -i ../hosts hw4-main.yml --ask-become --extra-vars "array_state='present'"
#или
ansible-playbook -i ../hosts hw4-main.yml --ask-become


#удаление массива
ansible-playbook -i ../hosts hw4-main.yml --ask-become --extra-vars "array_state='absent'"

#

# =============== после выполнения плейбука (создание массива) идем на хост по ssh
ssh otus-vm2

# =============== проверяем состояние массива после создания
ladm@otus-vm2:~$ sudo cat /proc/mdstat
	
		Personalities : [raid6] [raid5] [raid4] [raid0] [raid1] [raid10]
		md0 : active raid5 sdd[4] sdb[0] sdc[2] sdf[1]
			  3139584 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]

		unused devices: <none>


ladm@otus-vm2:~$ sudo mdadm -D /dev/md0
		/dev/md0:
				   Version : 1.2
			 Creation Time : Sun Mar 16 15:43:10 2025
				Raid Level : raid5
				Array Size : 3139584 (2.99 GiB 3.21 GB)
			 Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
			  Raid Devices : 4
			 Total Devices : 4
			   Persistence : Superblock is persistent

			   Update Time : Sun Mar 16 15:45:04 2025
					 State : clean
			Active Devices : 4
		   Working Devices : 4
			Failed Devices : 0
			 Spare Devices : 0

					Layout : left-symmetric
				Chunk Size : 512K

		Consistency Policy : resync

					  Name : otus-vm2:0  (local to host otus-vm2)
					  UUID : a3a5d98f:a5430886:e5a1394b:cf8f8507
					Events : 20

			Number   Major   Minor   RaidDevice State
			   0       8       16        0      active sync   /dev/sdb
			   1       8       80        1      active sync   /dev/sdf
			   2       8       32        2      active sync   /dev/sdc
			   4       8       48        3      active sync   /dev/sdd
# =========================================================================== 


# =============== фейлим один из дисков в массиве и проверяем состояние массива
ladm@otus-vm2:~$ sudo mdadm /dev/md0 --fail /dev/sdb
mdadm: set /dev/sdb faulty in /dev/md0

ladm@otus-vm2:~$ sudo mdadm -D /dev/md0
		/dev/md0:
				   Version : 1.2
			 Creation Time : Sun Mar 16 15:43:10 2025
				Raid Level : raid5
				Array Size : 3139584 (2.99 GiB 3.21 GB)
			 Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
			  Raid Devices : 4
			 Total Devices : 4
			   Persistence : Superblock is persistent

			   Update Time : Sun Mar 16 16:30:12 2025
					 State : clean, degraded
			Active Devices : 3
		   Working Devices : 3
			Failed Devices : 1
			 Spare Devices : 0

					Layout : left-symmetric
				Chunk Size : 512K

		Consistency Policy : resync

					  Name : otus-vm2:0  (local to host otus-vm2)
					  UUID : a3a5d98f:a5430886:e5a1394b:cf8f8507
					Events : 22

			Number   Major   Minor   RaidDevice State
			   -       0        0        0      removed
			   1       8       80        1      active sync   /dev/sdf
			   2       8       32        2      active sync   /dev/sdc
			   4       8       48        3      active sync   /dev/sdd

			   0       8       16        -      faulty   /dev/sdb

# =========================================================================== 


# =============== удаляем "поврежденный" диск, смотрим состояние массива

ladm@otus-vm2:~$ sudo mdadm /dev/md0 --remove /dev/sdb
mdadm: hot removed /dev/sdb from /dev/md0


ladm@otus-vm2:~$ sudo mdadm -D /dev/md0
		/dev/md0:
				   Version : 1.2
			 Creation Time : Sun Mar 16 15:43:10 2025
				Raid Level : raid5
				Array Size : 3139584 (2.99 GiB 3.21 GB)
			 Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
			  Raid Devices : 4
			 Total Devices : 3
			   Persistence : Superblock is persistent

			   Update Time : Sun Mar 16 16:30:33 2025
					 State : clean, degraded
			Active Devices : 3
		   Working Devices : 3
			Failed Devices : 0
			 Spare Devices : 0

					Layout : left-symmetric
				Chunk Size : 512K

		Consistency Policy : resync

					  Name : otus-vm2:0  (local to host otus-vm2)
					  UUID : a3a5d98f:a5430886:e5a1394b:cf8f8507
					Events : 23

			Number   Major   Minor   RaidDevice State
			   -       0        0        0      removed
			   1       8       80        1      active sync   /dev/sdf
			   2       8       32        2      active sync   /dev/sdc
			   4       8       48        3      active sync   /dev/sdd
# =========================================================================== 


# =============== добавляем новый чистый диск, дожидаемся окончания ребилда и смотрим итоговое состояние массива

ladm@otus-vm2:~$ sudo mdadm /dev/md0 --add /dev/sde
mdadm: added /dev/sde
ladm@otus-vm2:~$ sudo cat /proc/mdstat
		Personalities : [raid6] [raid5] [raid4] [raid0] [raid1] [raid10]
		md0 : active raid5 sde[5] sdd[4] sdc[2] sdf[1]
			  3139584 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/3] [_UUU]
			  [==============>......]  recovery = 70.6% (740348/1046528) finish=0.0min speed=148069K/sec

		unused devices: <none>
		
		
ladm@otus-vm2:~$ sudo cat /proc/mdstat
		Personalities : [raid6] [raid5] [raid4] [raid0] [raid1] [raid10]
		md0 : active raid5 sde[5] sdd[4] sdc[2] sdf[1]
			  3139584 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]

		unused devices: <none>
		
		
ladm@otus-vm2:~$ sudo mdadm -D /dev/md0
		/dev/md0:
				   Version : 1.2
			 Creation Time : Sun Mar 16 15:43:10 2025
				Raid Level : raid5
				Array Size : 3139584 (2.99 GiB 3.21 GB)
			 Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
			  Raid Devices : 4
			 Total Devices : 4
			   Persistence : Superblock is persistent

			   Update Time : Sun Mar 16 16:31:39 2025
					 State : clean
			Active Devices : 4
		   Working Devices : 4
			Failed Devices : 0
			 Spare Devices : 0

					Layout : left-symmetric
				Chunk Size : 512K

		Consistency Policy : resync

					  Name : otus-vm2:0  (local to host otus-vm2)
					  UUID : a3a5d98f:a5430886:e5a1394b:cf8f8507
					Events : 42

			Number   Major   Minor   RaidDevice State
			   5       8       64        0      active sync   /dev/sde
			   1       8       80        1      active sync   /dev/sdf
			   2       8       32        2      active sync   /dev/sdc
			   4       8       48        3      active sync   /dev/sdd
# =========================================================================== 