# unsafe-long-to-long-map
Primitive based map implemented with sun.misc.Unsafe class
```java
import sun.misc.Unsafe;

import java.util.Objects;

import static java.lang.String.format;

/**
 * Замечания по заданию:
 *
 *  1. Из текста задания делаю вывод, что о непосредственном выделении памяти заботится внешний аллокатор,
 *  соответственно, вызов методов sun.misc.Unsafe.setMemory(long, long, byte) и sun.misc.Unsafe.freeMemory(long)
 *  отдается на откуп пользователям класса, в противном случае класс будет нарушать SCP.
 *
 *  2. В тексте задания нет никаких требований касательно работы класса в многопоточном режиме (более того,
 *  в задании указано, что методов sun.misc.Unsafe.getLong(long) и sun.misc.Unsafe.putLong(long, long))
 *  достаточно для выполнения задания. Соответственно, в текущей реализации данный класс не является потокобезопасным.
 *  
 *  3. По причине отсутствия технической возможности протестировать класс на ~100Gb выделенной памяти класс протестирован на ~10Gb выделенной памяти.
 */

/**
 * Требуется написать LongLongMap который по произвольному long ключу хранить произвольное long значение
 * Важно: все данные (в том числе дополнительные, если их размер зависит от числа элементов) требуется хранить в выделенном заранее блоке в разделяемой памяти, адрес и размер которого передается в конструкторе
 * для доступа к памяти напрямую необходимо (и достаточно) использовать следующие два метода:
 * sun.misc.Unsafe.getLong(long), sun.misc.Unsafe.putLong(long, long)
 */
public class LongLongMap {
    private final static long MIN_MAP_SIZE = 8L;
    private final static long LONG_SIZE = 8L;
    private final Unsafe unsafe;
    private final long address;
    private final long size;
    private final long capacity;

    /**
     * @param unsafe  для доступа к памяти
     * @param address адрес начала выделенной области памяти
     * @param size    размер выделенной области в байтах (~100GB)
     */
    LongLongMap(Unsafe unsafe, long address, long size) {
        if (size < MIN_MAP_SIZE) {
            throw new IllegalArgumentException(format("Map size %sb is less than minimal size %sb.", size, MIN_MAP_SIZE));
        }
        if (size % LONG_SIZE != 0) {
            throw new IllegalArgumentException(format("Map size %sb is not divisible by %sb.", size, LONG_SIZE));
        }
        this.unsafe = Objects.requireNonNull(unsafe);
        this.address = address;
        this.size = size;
        this.capacity = size / LONG_SIZE;
    }

    /**
     * Метод должен работать со сложностью O(1) при отсутствии коллизий, но может деградировать при их появлении
     *
     * @param k произвольный ключ
     * @param v произвольное значение
     * @return предыдущее значение или 0
     */
    public long put(long k, long v) {
        long addr = getValueAddressByKey(k), pv = getValueByAddress(addr);
        unsafe.putLong(addr, v);
        return pv;
    }

    /**
     * Метод должен работать со сложностью O(1) при отсутствии коллизий, но может деградировать при их появлении
     *
     * @param k ключ
     * @return значение или 0
     */
    public long get(long k) {
        return getValueByAddress(getValueAddressByKey(k));
    }

    /**
     * Получение значения типа long по его адресу
     *
     * @param addr ключ
     * @return значение типа long указанному адресу
     */
    private long getValueByAddress(long addr) {
        return unsafe.getLong(addr);
    }

    /**
     * Получение индекса в хэш таблице для произвольного ключа
     *
     * @param k произвольный ключ
     * @return индекс ключа в хэш таблице
     */
    private long getKeyIndex(long k) {
        return Math.abs(k) % capacity;
    }

    /**
     * Получение адреса значения в хэш таблице для произвольного ключа
     *
     * @param k произвольный ключ
     * @return адрес значения в хэш таблице для произвольного ключа
     */
    private long getValueAddressByKey(long k) {
        return address + getKeyIndex(k) * LONG_SIZE;
    }
}
```
